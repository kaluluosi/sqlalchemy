=============
0.9版本更新日志
=============

.. changelog_imports::

    .. include:: changelog_08.rst
        :start-line: 5


    .. include:: changelog_07.rst
        :start-line: 5


.. _未发布的changelog ::
    :version: 0.9.11

    .. change::
        :tags: bug, oracle, py3k
        :tickets: 3491
        :versions: 1.0.9

        修复了cx_Oracle版本5.2的支持，该版本在Python 3下会触发SQLAlchemy版本检测错误，并无意中未使用正确的Unicode模式。这会导致绑定变量被错误解释为空和行未被返回等问题。

    .. change::
        :tags: bug, engine
        :tickets: 3497
        :versions: 1.0.8

        修复了关键性问题，即池中的“checkout”事件处理程序可能会被调用无效连接而未被“connect”事件处理程序调用，在池被使无效并失败重新连接后。无效连接将保持存在，将在下一次重试时使用。这个问题对1.0系列更大的影响在于1.0.2之后，因为它还将带有空白的“ .info”字典传递到事件处理程序中;在1.0.2之前，“ .info”字典仍然是以前的字典。

.. changelog ::
    :version: 0.9.10
    :released: July 22, 2015

    .. change::
        :tags: bug, sqlite
        :tickets: 3495
        :versions: 1.0.8

        修复了SQLite方言的错误，该错误导致非字母字符（如点或空格）包含在唯一约束名称中的反射不会反射其名称。

    .. change::
        :tags: feature, sql
        :tickets: 3418
        :versions: 1.0.5

        在  :meth:`_expression.Insert.from_select`   中添加了官方支持子查询中使用的CTE。在0.9.9之前，该行为是偶然发生的，当时更改了与:ticket：'3248'相关联的内容。请注意，这是将WITH子句呈现为INSERT之后，SELECT之前的内容。在INSERT，UPDATE，DELETE的顶层呈现CTE的全部功能是针对以后的版本的新功能。

    .. change::
        :tags: bug, ext
        :tickets: 3408
        :versions: 1.0.4

        修复了使用扩展属性仪器系统时会发生的错误，当  :meth:`.class_mapper`  使用无效的输入调用并且也不是弱引用时（如整数）时，将无法正确引发异常。

    .. change::
        :tags: bug, tests, pypy
        :tickets: 3406
        :versions: 1.0.4

        修复了一个导入问题，该导入问题阻止了“pypy setup.py test”正常工作。

    .. change::
        :tags: bug, engine
        :tickets: 3375
        :versions: 1.0.1

        向  :paramref:`_pool.Pool.reset_on_return`  参数接受的字符串值中添加了字符串值“none”作为“None”的同义词，因此可以使用字符串值来使用诸如 :func:` .engine_from_config`之类的实用程序而不会出现问题。

    .. change::
        :tags: bug, sql
        :tickets: 3362
        :versions: 1.0.0

        修复了使用命名约定的 :class:`_schema.MetaData` 对象不能正确使用pickle的问题。跳过了该属性导致从中基于其他表创建额外表时出现不一致和失败的情况。

    .. change::
        :tags: bug, postgresql
        :tickets: 3354
        :versions: 1.0.0

        修复了长期存在的错误，即在将 :class:`.Enum` 类型与psycopg2方言一起使用且非ASCII值和“native_enum = False”结合使用时，无法正确解码返回结果。这源自于PG: class:`_postgresql.ENUM`类型曾经是没有“非本地”选项的独立类型。

    .. change::
        :tags: bug, orm
        :tickets: 3349

        在使用  :meth:`_query.Query.update`  或  :meth:` _query.Query.delete`  方法时，  :class:`_query.Query`  或  :meth:` _query.Query.select_from`  之类的方法，则会发出警告而不是静默地忽略这些字段，并且自1.0.0b5起，这将引发错误。

    .. change::
        :tags: bug, orm
        :tickets: 3352
        :versions: 1.0.0b5

        修复了多个嵌套  :meth:`.Session.begin_nested`  操作内的状态跟踪将无法传播在内部保存点中已经更新的对象的"dirty"标志的错误，因此，如果回滚封闭保存点，则该对象将不是过期状态的一部分，因此会被还原为其数据库状态。

    .. change::
        :tags: bug, mysql, pymysql
        :tickets: 3337
        :versions: 1.0.0b4

        修复了PyMySQL的Unicode支持，当使用unicode参数的"executemany"操作时。SQLAlchemy现在传递语句以及绑定参数作为Unicode对象，因为PyMySQL通常在内部使用字符串插值来生成最终语句，在执行多个操作时只执行"编码"步骤最后的语句。

    .. change::
        :tags: bug, py3k, mysql
        :tickets: 3333
        :versions: 1.0.0b2

        修复了Py3K上 :class:`.mysql.BIT` 类型未正确使用“ord（）”函数的错误。David Marin提供了请求拉取。

    .. change::
        :tags: bug, ext
        :tickets: 3324

        修复了0.9.9中从sqlalchemy.ext.declarative中删除 :func:`.as_declarative` 符号的回归。

    .. change::
        :tags: feature, orm
        :tickets: 3320
        :versions: 1.0.0b1

        :attr:`_query.Query.column_descriptions` 中添加了一个新条目“entity”的字典。这是指由表达式引用的主ORM映射类或别名类。与现有的“type”条目相比，它将始终是一个映射实体，即使是从列表达式中提取的，或者是纯核心表达式的情况下也是如此。如果在0.9.10中未发布的定义中次要分类，此功能的:ticket：'3403'也发生了回归。

.. changelog ::
    :version: 0.9.9
    :released: March 10, 2015

    .. change::
        :tags: feature, postgresql
        :versions: 1.0.0b1

        在使用PostgreSQL索引时添加了“CONCURRENTLY”关键字的支持，该索引使用“postgresql_concurrently”进行建立。Iuri de Silvio提供了请求拉取。

        .. seealso::

              :ref:`postgresql_index_concurrently` 

    .. change::
        :tags: bug, ext, py3k
        :versions: 1.0.0b1

        修复了关联代理列表类在Py3K下无法正确解释片段的错误。Gilles Dartiguelongue 提供了请求拉取。

    .. change::
        :tags: feature, sqlite
        :versions: 1.0.0b1

        在SQLite上添加了对部分索引（例如使用WHERE子句）的支持。Kai Groner提供了请求拉取。

        .. seealso::

              :ref:`sqlite_partial_index` 

    .. change::
        :tags: bug, orm
        :tickets: 3310
        :versions: 1.0.0b1

        修复了ORM对象比较中的错误，其中如果源是别名类，则连接的多对一“！= None”比较将失败，或者如果查询需要由于别名连接或多态查询而应用特殊别名，则会失败;还修复了将连接的多对一比较与对象状态进行比较的情况，如果查询需要由于别名连接或多态查询而应用特殊别名，则会失败。

    .. change::
        :tags: bug, orm
        :tickets: 3309
        :versions: 1.0.0b1

        修复了在  :class:`.Session` .Session` 添加状态，并且任务警告和删除此状态(由:ticket:`2389` )会尝试继续时，将引发TypeError的情况。正确的行为是放弃添加状态，要么是在检测到这样的添加之后，要么是在ISessionTransaction`.`rollback was called之后。

    .. change::
        :tags: bug, orm
        :tickets: 3300
        :versions: 1.0.0b1

        修正懒加载SQL构建的问题，即当一个引用自身的连接多次引用了相同的“本地”列时，多次引用相同的“本地列”时，主键连接可能无法在所有情况下进行替换。为确定此类替换，重新设计了用于检测替换的逻辑以更具开放性.

    .. change::
        :tags: bug, postgresql
        :tickets: 2940
        :versions: 1.0.0b1

        在与psycopg2方言一起使用PG `UUID`类型的情况下，修复了与ARRAY类型的兼容性问题。psycopg2方言现在使用psycopg2.extras.register_uuid()钩子，以便始终将UUID值作为UUID()对象传递/从DBAPI传递。  :paramref:`.UUID.as_uuid`  标志仍然受到尊重，但是，如果禁用了返回的UUID对象，则需要将其转换回字符串。

    .. change::
        :tags: bug, postgresql
        :versions: 1.0.0b1

        在使用psycopg2 2.5.4或更高版本时，为  :class:`postgresql.JSONB` ` json_deserializer``时建立了传递给方言的JSON反序列化器。还修复了实际上没有往返JSONB类型而不是JSON类型的PostgreSQL集成测试。Mateusz Susik提供了请求拉取。

    .. change::
        :tags: bug, postgresql
        :versions: 1.0.0b1

        在使用早期版本的psycopg2 <2.4.3时向HSTORE类型注册“ array_oid”标志时，修复了HSTORE类型的使用以及远程侧通过FK传输中的列类型的问题，该问题会在目标列具有与其名称不同的键值且使用link_to_name时引用时。 。

    .. change::
        :tags: bug, orm
        :tickets: 3287

        增加了复杂自引用primaryjoin包含函数时发出警告的功能，同时还指定了remote_side；如果remote_side存在，则只在remote_side不存在时发出警告。

    .. change::
        :tags: bug, sql
        :versions: 1.0.0b1
        :tickets: 3248

        修复了将SELECT嵌入到INSERT中时，例如通过值子句或作为“from select”时，列可能会污染在由RETURNING子句产生的结果集中使用的列类型的问题。当两个语句的列共享相同的名称时，这可能导致错误或误解适应返回的行时。

    .. change::
        :tags: bug, orm
        :versions: 1.0.0b1
        :tickets: 3241

        修复了表和索引反射的调整，以便如果索引报告在表中不存在的列，则会发出警告并跳过该列。对于一些特殊的系统列情况，这可能会在Oracle中发生，观察到此问题后。

    .. change::
        :tags: bug, ext
        :versions: 1.0.0b1

        修复了 :class:`.ext.mutable.MutableDict` 未实现“update（）”字典方法的错误，因此未捕获更改。Matt Chisholm提供了请求拉取。

    .. change::
        :tags: bug, ext
        :versions: 1.0.0b1

        修复了无法在“强制”操作中看到  :class:`.ext.mutable.MutableDict` .ext.mutable.MutableDict` 而不是进行“虎变”操作。Matt Chisholm 提供了请求拉取。

    .. change::
        :tags: bug, pool
        :versions: 1.0.0b1
        :tickets: 3168

        修复了连接池日志记录中的“connection checked out”调试记录消息不会发出的问题，如果日志设置使用“ logging.setLevel()”，而不是使用“echo_pool”标志。已添加测试以断言此记录。

    .. change::
        :tags: feature, postgresql, pg8000
        :versions: 1.0.0b1

        支持使用pg8000驱动程序的“ sane multi row count”，这主要适用于使用ORM的版本控制。该功能在使用pg8000 1.9.14或更高版本检测到版本。Tony Locke提供了请求拉取。

    .. change::
        :tags: bug, engine
        :versions: 1.0.0b1
        :tickets: 3165

        在插入或更新受影响的字符串键组成“compiled cache”缓存键时，现在对它们进行排序。在此前，这些键先前不可确定排序，这意味着相同的语句可能会基于等效密钥多次缓存，这些缓存会在内存和性能方面代价高昂。

    .. change::
        :tags: bug, orm
        :versions: 1.0.0b1
        :tickets: 3194

        修复了一个潜在的警告，即，在复杂的自引用primaryjoin包含函数时，如果同时指定了remote_side，那么将发出必须设置remote_side的警告。现在仅在remote_side不存在时发出警告。

    .. change::
        :tags: bug, postgresql
:versions: 1.0.0b1
        :tickets: 3159

        修复了PostgreSQL JSON类型无法保留或以其他方式呈现SQL NULL列值的错误，而不是JSON编码的“null”的错误。为支持此情况，更改如下：

        *现在可以指定值：func:`.null`，这将始终导致结果语句中的NULL值。

        *添加了一个新的参数  :paramref:`_types.JSON.none_as_null`  ，当为True时指示应将Python“None”值持久化为SQL NULL，而不是JSON编码的“null”。

        检索NULL作为None也已被修复，而不是使用psycopg2之外的DBAPI，即pg8000。

    .. change::
        :tags: bug, sql
        :versions: 1.0.0b1
        :tickets: 3154

        修复了CTE中存在的bug，其中“literal_binds”编译器参数将在一个CTE中引用另一个别名CTE时，不会始终正确传播。

    .. change::
        :tags: bug, postgresql
        :versions: 1.0.0b1
        :tickets: 3075

        现在，DBAPI错误的异常包装系统可以容纳非标准的DBAPI异常，例如psycopg2 TransactionRollbackError。这些异常现在将使用最接近的可用子类在“sqlalchemy.exc”中抛出，在TransactionRollbackError的情况下为“sqlalchemy.exc.OperationalError”。

    .. change::
        :tags: bug, sql
        :versions: 1.0.0b1
        :tickets: 3144, 3067

        修复了0.9.7中由于：ticket：'3067'引起的回归，以及所谓的“schema”类型，例如：class：`。Boolean`和：class：'Enum'不能再被pickled的多命名单元测试等问题。

    .. change::
        :tags: bug, postgresql
        :versions: 1.0.0b1
        :tickets: 3141

        修复了 :class:`_postgresql.array` 对象中的bug，其中与普通的Python列表进行比较会失败，而不能使用正确的数组构造函数。由Andrew发起的拉取请求。

    .. change::
        :tags: bug, postgresql
        :versions: 1.0.0b1
        :tickets: 3137

        向函数（例如“func”构造）添加了支持的：meth:`.FunctionElement.alias` 方法，例如添加了一个`with the given alias name`参数的函数。以前，此方法的行为是未定义的。当前行为模仿了0.9.4之前的行为，即将函数转换为具有给定别名名称的单列FROM条款，其中列本身是匿名命名的。

.. changelog::
    :version: 0.9.7
    :released: July 22, 2014

    .. change::
        :tags: bug, postgresql, pg8000
        :tickets: 3134
        :versions: 1.0.0b1

        修复了0.9.5中由新pg8000隔离级别功能引起的问题，即在连接上引发引擎级隔离级别参数将在连接上引发错误。

    .. change::
        :tags: bug, oracle, tests
        :tickets: 3128
        :versions: 1.0.0b1

        修复了oracle dialect测试套件中的bug，在一个测试中，“username”被假定在数据库URL中，即使这可能不是案例。

    .. change::
        :tags: bug, orm, eagerloading
        :tickets: 3131
        :versions: 1.0.0b1

        修复了由ticket：'2976'引起的回归，该回归在0.9.4中发布，其中沿着加入了eager加载的链的一个“外部连接”传播会不正确地将同级加入路径上的“内部连接”也转换为“外部连接”，当仅后代路径应该接收“外部连接”传播时，另外，修复了“嵌套式”连接在两个兄弟连接路径之间不适当地发生的相关问题。

    .. change::
        :tags: bug, sqlite
        :tickets: 3130
        :versions: 1.0.0b1

        修复了SQLite连接重写问题，其中作为标量子查询嵌入的子查询，例如在IN中使用的嵌入式子查询，会从包含查询中接收不适当的替换，如果在子查询中与包含查询相同的表格出现，则很可能减少此问题，例如在加入继承场景中。

    .. change::
        :tags: bug, sql
        :tickets: 3067
        :versions: 1.0.0b1

        修复了命名约定功能中的错误，其中使用了包括“constraint_name”的检查约束约定，然后将强制所有  :class:`.Boolean` .Enum` 也需要名称，因为它们隐含地创建一个约束条件，即使最终目标后端是一个不需要生成约束的后端，例如PostgreSQL。对于这些特定约束的命名约定的机制已重新组织，使得命名确定是在DDL编译时完成的，而不是在约束/表构建时间完成的。

    .. change::
        :tags: bug, mssql
        :tickets: 3025

        修复了来自0.9.5的回归，该回归由：ticket：3025引起，其中用于确定“默认模式”的查询在SQL Server 2000中无效。对于SQL Server 2000，我们回到默认为'schema name'参数的“schema name”，该参数可配置但默认为'dbo'。

    .. change::
        :tags: bug, orm
        :tickets: 3083, 2736
        :versions: 1.0.0b1

        由于：ticket：'2736'引起的0.9.0中的回归，对于  :meth:`_query.Query.select_from`  方法现在不再设置查询的“from entity”属性正确，因此在使用字符串名称搜索属性时可能无法检查适当的“from”实体。

    .. change::
        :tags: bug, sql
        :tickets: 3090
        :versions: 1.0.0b1

        修复了常用表达式中的bug，其中在某些方式中嵌套CTE时，可能会导致位置绑定参数以错误的最终顺序表示。

    .. change::
        :tags: bug, sql
        :tickets: 3069
        :versions: 1.0.0b1

        修复了数值为0的多值：class:`_expression.Insert`构造的错误，该构造方式可能会失败，方法是对于文本SQL表达式给出的第一个以外的后续值条目未进行检查。

    .. change::
        :tags: bug, sql
        :tickets: 3123
        :versions: 1.0.0b1

        在语法中添加了“str()”步骤，以遍历Python版本<2.6.5的dialect_kwargs，解决了过程中传递这些参数的问题，“no unicode keyword arg”错误，因为这些参数作为关键词参数在某些反射过程中传递。

    .. change::
        :tags: bug, sql
        :tickets: 3122
        :versions: 1.0.0b1

         :meth:`.TypeEngine.with_variant`  long established 的同样惯例。

    .. change::
        :tags: bug, orm
        :tickets: 3117

        query.update() /delete()的“评估器”将无法处理多表，需要将其设置为`synchronize_session = False`或`synchronize_session ='fetch'`；现在发出警告。在1.0中，这将提升为一个完整的异常。

    .. change::
        :tags: bug, tests
        :versions: 1.0.0b1

        修复了“python setup.py test”未正确调用distutils的错误，此错误将在测试套件结束时发出错误。

    .. change::
        :tags: feature, postgresql
        :versions: 1.0.0b1
        :tickets: 3078

        为PostgreSQL方言添加了一个新标志：paramref:`_types.ARRAY.zero_indexes`。当设置为True时，会向所有数组索引值添加一个值，然后将其传递给数据库，从而允许Python样式的基于零的索引和PostgreSQL基于一的索引之间更好地互操作。Pull request courtesy Alexey Terentev。

    .. change::
        :tags: feature, postgresql
        :versions: 1.0.0b1

        通过 :class:`_postgresql.JSONB` 添加了对PostgreSQL JSONB的支持。Pull request courtesy Damian Dimmich。

    .. change::
        :tags: feature, mssql
        :versions: 1.0.0b1

        为SQL Server 2008启用了“multivalues insert”。通过Albert Cervin的拉取请求。还扩展了关于“IDENTITY INSERT”模式的检查，以包括当标识键存在于VALUEs子句中时的情况。

    .. change::
        :tags: feature, engine
        :tickets: 3076
        :versions: 1.0.0b1

        添加了新事件：meth:`_events.ConnectionEvents.handle_error`，这是一个更全面和全面的`_events.ConnectionEvents.dbapi_error`的替代方法。

    .. change::
        :tags: bug, orm
        :tickets: 3108
        :versions: 1.0.0b1

        修复了在savepoint块中持久化、删除或主键更改的item不会在回滚更外层事务后恢复其原始状态（不在会话中，在会话中，以前的PK）的问题。

    .. change::
        :tags: bug, orm
        :tickets: 3106
        :versions: 1.0.0b1

        修复了与 :func:`.with_polymorphic` 结合使用的子查询急切加载的问题，其中子查询负载的实体和列的定向已与此类型的实体和其他实现更准确。

    .. change::
        :tags: bug, orm
        :tickets: 3099

        修复了动态属性的错误，这是：ticket：'3060'的回归，其中具有lazy ='dynamic'的自我参照关系将在刷新操作中引发TypeError。

    .. change::
        :tags: bug, declarative
        :tickets: 3097
        :versions: 1.0.0b1

        修复了未将declarative“__abstract__”标志与实际值“False”进行区分的错误。在进行测试等级时，表明“__abstract__”标志需要实际评估为True。

.. changelog::
    :version: 0.9.6
    :released: June 23, 2014

    .. change::
        :tags: bug, orm
        :tickets: 3060

        取消了对  :ticket:`3060`  的更改--这是针对单元的修补程序
        工作，最终在1.0中通过:ticket：`3061`更全面地更新。
        定位于由UPDATE转换为DELETE/INSERT的情况中由于lazy
        load可能产生事件的问题。

.. changelog::
    :version: 0.9.5
    :released: June 23, 2014

    .. change::
        :tags: bug, orm
        :tickets: 3042
        :versions: 1.0.0b1

        添加了一个新参数：paramref:`.orm.mapper.confirm_deleted_rows`。默认值为True，表示一系列DELETE语句应确认游标
        行数与应该匹配的主键数量匹配；这种行为已从大多数案例中取出
        （除非使用version_id），以支持不同寻常的极端情况              自我参照ON DELETE CASCADE；为适应这种情况，
        消息现在只是警告而不是异常，而标志可以被使用                                       表示预期自我参照级联删除的映射。
        另请参见：ticket：`2403`了解背景。

    .. change::
        :tags: bug, ext, automap
        :tickets: 3004

        添加了对于自动映射的支持，其中不应创建两个类之间的关系在连接继承关系的情况下，对于将子类链接回超类的那些外键，这些外键将链接回超类。

    .. change::
        :tags: bug, orm
        :tickets: 2948

        修复了一个非常旧的行为，即可以适当满足延迟加载的一个-to-many会在不同于父表的限定符包括某些种类的限定符时，错误地拉动父表并根据父表的内容返回结果，例如，当primaryjoin包含一些针对父表的鉴别器时，如`and_(parent.id == child.parent_id, parent.deleted == False)`。
        虽然这个primaryjoin不那么适合一个到多的，但是当通过backref的方法获得的许多到一个之后，它稍微常见一些。

    .. change::
        :tags: bug, orm
        :tickets: 2965

        改进了“如何从A加入B”的检查，使得当一个表很多时，复合外键针对父表，突出：paramref:`_orm.relationship.foreign_keys`参数将正确解释，以解决歧义；之前，此状态将引发多个FK路径，当事实上需要外键参数来确定路径时，foreign_keys参数应该建立哪一个。

    .. change::
        :tags: bug, mysql

        调整了mysql-connector-python的设置；在Py2K中，“支持unicode语句”的标志现在为False，因此SQLAlchemy将编码*SQL字符串*（请注意：*不是*参数）
        到字节，然后发送到数据库。这似乎允许mysql-connector中所有与Unicode相关的测试都通过，包括使用非ASCII表/列名称以及在
        使用unicode在cursor.executemany()下使用TEXT类型的某些测试。

    .. change::
        :tags: feature, engine

        为方言级事件添加了一些新事件机制；首次实现允许事件处理程序重新定义由特定方言在DBAPI游标上调用execute()或executemany（）的具体机制。这些新事件，此时半公开和试验性，是为了支持一些即将推出的事务相关扩展。

    .. change::
        :tags: feature, engine
        :tickets: 2978

        可以在  :class:`_engine.Engine` .Session` 或通过显式连接）并且侦听器将从这些连接中获取事件。以前，性能问题使事件转移从 :class:`_engine.Engine` 到：class:`_engine.Connection`请勿随意打开事件转移。

    .. change::
        :tags: bug, tests
        :tickets: 2980

        修复了一些放置在Py3.2中妨碍测试通过的错误的“errant u”字符串。补丁Courtesy Arfrever Frehtes Taifersar Arahesis.

    .. change::
        :tags: bug, engine
        :tickets: 2985

        改进了 :class:`_engine.Engine` 在检测到“断开”条件时如何回收连接池的机制；现在，而不是丢弃池并明确关闭连接，
        池被保留并更新为反映当前时间的“生成”时间戳，从而导致在下一次它们被检查时，将回收所有现有连接。这极大地简化了回收过程，
        消除了等待旧池的连接尝试的“唤醒”并消除了可能立即丢弃“池”对象的竞争条件             在回收操作期间创建。

    .. change::
        :tags: bug, oracle
        :tickets: 2987

        添加了新的datatype   :class:`_oracle.DATE` ，它是 :class:` .DateTime`的子类。由于Oracle没有“datetime”类型，因此它只有“DATE”，因此在尽可能运行的情况下，将日期类型强制转换而来。.. changelog::
    :version: 0.9.1
    :released: January 15, 2014

    .. change::
        :tags: mysql, feature
        :tickets: 2902

        Added a new dialect-level create_engine argument,
        ``client_flag``, which allows the specification of a series of flags
        to be sent to MySQL at connection time via the
        ``client_flag`` parameter of ``CONNECT_ATTRS``, a feature supported
        by all MySQL DBAPIs.  Pull request courtesy Mikhail Korobov.

    .. change::
        :tags: feature, sql

        Added  :meth:`_engine.result.ResultProxy.scalars`  to allow more
        convenient access to a simple column scalar value from a single-row
        result set.

    .. change::
        :tags: bug, orm, sqlite
        :tickets: 2898

        Fixed bug whereby relationship attributes would fail to work properly
        for a joined-inheritance subclass when the subclass local column
        name did not match that of the parent on the right-hand-side of the
        relationship.

    .. change::
        :tags: bug, mysql, dialect

        Fixed issue reintroduced in 0.8 whereby no result row was returned from
        "SELECT 1" by the MySQL dialect due to a difference in how MySQL-Python
        versus PyMySQL handle result set iteration.

    .. change::
        :tags: bug, orm

        Fixed regression in 0.9 where with_polymorphic eager loads off of
        an empty set would return no rows, as opposed to all rows.

    .. change::
        :tags: feature, orm

        Added support for "skipping" a  :paramref:`_orm.relationship`  loader for a specific
        row, most specifically for the purpose of "bulk refreshing" against
        identity map-enabled sessions.  Usage is to call
         :meth:`_orm.session.SkipLoader.populate_existing` ,
        passing the primary key value(s), e.g.::

            session.query(MyClass).populate_existing([1, 2, 3])

        versions back to 0.8.2.

    .. change::
        :tags: feature, sql
        :tickets: 2899

        Added support for common table expressions (CTEs) to the ORM's
         :meth:`_query.Query.cte`  method which generates a   :class:` _expression.Alias`  object
        with a contained   :class:`_expression.Select`  rendered as a "WITH ..."
        clause.  The new method also returns the   :class:`_expression.Alias`  object
        such that it can be used in filtering, ordering, and included
        in other SELECT statements.  A fourth example is added to the
        ORM tutorial to illustrate this.

    .. change::
        :tags: feature, sql

        Added a new function   :func:`_engine.result.RowProxy._fields`  which
        returns the column keys of the enclosing   :class:`_engine.result.RowProxy` .

    .. change::
        :tags: feature, sql

        Added the ability for   :class:`_expression.Annotated`  to key expressions
        off of other expressions, rather than just on columns.  E.g.::

            from sqlalchemy.sql import column, text
            from sqlalchemy.sql.expression import Annotated

            # prefix a string to an expression
            stmt = select([Annotated(text("'foo'"), column('a'))])

    .. change::
        :tags: feature, sql

        Added a new module-level function   :func:`_expression.literal_column` 
        which generates a   :class:`_expression.ColumnClause`  object that
        is unsupported for rendering within any SQL expressions except
        for those that render plain column references.

    .. change::
        :tags: feature, sql

        Added a new   :func:`_expression.type_coerce`  function which applies
        a CAST expression to an expression in order to coerce it to a new type.

    .. change::
        :tags: bug, ext

        Fixed regression whereby the   :class:`.AssociationProxy`  class did
        not work with the new features of partial mapping and dictionaries.

    .. change::
        :tags: feature, ext

        Added convenience methods to   :class:`.AssociationProxy`  for
        generating association objects from single-attribute references,
        as well as a new "proxy dict" mode that allows the value of the
        proxied attribute to be accessed like a key/value mapping.

    .. change::
        :tags: bug, orm

        Corrected an issue whereby a joined-table-inheritance mapper that
        referred to a subclass in a relationship would use local table-aliasing
        for the subclass in the generated SQL instead of that of the subclass.

    .. change::
        :tags: bug, orm

        Fixed regression inadvertently introduced in 0.9 whereby a flush
        against a model with a "dynamic" relation and with a "passive"
        foreign key constraint would fail due to the eager loader created
        for the relation not issuing a SELECT statement.

    .. change::
        :tags: feature, orm

        Implemented so-called "partial-column-based" inheritance mapping,
        which allows a subclass to be defined based on only a subset of the
        columns present within the parent table(s); columns not present
        in the subclass are not fetched from the parent table.

        Partial-column based inheritance has many important optimizations that
        can be used for mapping on certain underlying schemas, such as
        column-store databases.

        .. seealso::

              :ref:`examples_inheritance_partial` 

    .. change::
        :tags: feature, orm

        Parent tables for joined-table inheritance can now be fully specified
        using either the   :class:`_schema.Table`  object or a class, using
        the standard class:table syntax used by  :paramref:`_orm.relationship` 
        and other arguments.  Previously the class specification would
        always be applied to the secondary table.

    .. change::
        :tags: feature, orm

        Support added for polymorphic mapping against single- and
        multi-table inheritance discriminators as subject to order_by
        and limit/offset.  This allows both specificity- and grouping-based
        polymorphic queries to work with those additional clauses.

    .. change::
        :tags: bug, orm

        Fixed regression introduced in 0.9.0 whereby a parameter-based
        lazy loader with a many-to-one relationship would fail.

    .. change::
        :tags: feature, sql

        Added support for using   :class:`_expression.Literal`  within UNION, both in
        the SELECT list as expressions as well as within the ROW and
        VALUE expressions of UNION.

    .. change::
        :tags: feature, orm

        Added a new construct   :class:`_sql.Selectable`  which may be used in
        any context that a table can be used.   It represents a SELECT or other
        selectable construct that has no FROM clause; it is thus typically
        used with any number of "scalar select" queries as well as as a
        simple wrapper around subqueries or literal SQL.

          :class:`_sql.Selectable`  features a number of enhancements over using
        text("SELECT ...") for cases where a SELECT expression needs to be
        embedded within a larger statement: it provides enhanced compatibility
        with ORM queries as well as with core constructs such as INSERT
        and UPDATE statements with embedded SELECTs, and includes robustness
        in the generation of column labels.

    .. change::
        :tags: bug, orm

        Fixed an issue regarding LIMIT/OFFSET and ORM eager loading
        of collections which expand to non-literal SQL when appended
        to a SELECT such as a subquery or UNION.

    .. change::
        :tags: feature, orm

        Added support for ORM objects to be refreshed based on a secondary
        SQL expression rather than the default SELECT of the primary table.

        This is accommodated using a new parameter ``_refresh_expire_via_joins``
        which is accepted by  :meth:`_orm.mapper`  or at the class level via
          :class:`_orm.Declarative` , and is passed a SQL expression used
        to locate rows to be refreshed, as well as an optional dictionary
        of parameter/argument pairs that will supply values to the expression.

    .. change::
        :tags: bug, orm

        Fixed regression introduced in 0.9.0 whereby NULL were interpreted
        as being truthy when using  :meth:`_query.Query.exists` .

    .. change::
        :tags: feature, orm

        Added a new method  :meth:`_orm.attributes.QueryableAttribute.cmp_op` 
        which generates comparison operations for a fixed operator.
        E.g.::

            from sqlalchemy.orm import column_property

            class MyClass(Base):
                x = Column(Integer)
                y = column_property(x*2)
                z = column_property(x/3)

            cmp_op = MyClass.y.cmp_op(MyClass.z, '>')
            print session.query(MyClass).filter(cmp_op).all()

        The above query prints objects for which "x * 2 > x / 3".

    .. change::
        :tags: feature, orm

        A new  :paramref:`.relationship.order_by`  value, "random", is available
        which produces a new "ORDER BY rand()" directive on the related table each
        time the relationship is loaded.
        
        .. seealso::

              :ref:`relationship_orderby_random` 
                    

.. changelog::
    :version: 0.9.0
    :released: November 14, 2013

    After a year and a half of development, it is our pleasure to announce
    the release of SQLAlchemy 0.9.  This release is made up of over thousands
    of individual commits, hundreds of bug reports, pull requests, and often
    very helpful discussions.

    Major changes between 0.8 and 0.9 include the "ON CONFLICT" and
    common table expression support, index-only and partial indexes on
    Postgresql, "calling conventions" for stored procedures, customizable
    naming conventions for schema items, support for "returning" on
    MySQL and MariaDB, faster fetch speed for buffered result sets, as well
    as rewritten implementation of the ORM's "mapper" system with added
    support for ORM attribute events and "mapper versioning".

    A complete list of changes and ticket numbers can be found at:

    https://bitbucket.org/zzzeek/sqlalchemy/issues?milestone=0.9

    .. deprecated:: 0.9
        Support for SQL Server 2000 and Sybase has ended.  While it may
        still work against 0.9, testing of these platforms against major
        releases will no longer be performed.

        Support for non-Sphinx documentation formats is deprecated in this
        release, and will be removed in SQLAlchemy 1.0.

    .. deprecated:: 0.9
        The   :func:`.orm.reconstruct_instance()`  function has been marked as
        deprecated, with the given alternate instruction to make use of
        more specific means of creating objects which produce the needed
        state, such as   :func:`.attributes.flag_modified()`  and other
        modifier functions.

    .. deprecated:: 0.9
        The ability to set the database name using URL parameters (e.g.
        ``postgresql+psycopg2://localhost/dbname?dbname=mydb``) is deprecated,
        pass it as a positional argument instead:
        ``postgresql+psycopg2://localhost/mydb``.

    .. change::
        :tags: feature, orm
        :tickets: 1153
        :milestone: 0.9.0

        The ORM "mapper" configuration system has been almost completely
        rewritten, with significant streamlining and an emphasis towards
        allowing even more user-defined behavior during attribute mapping.

        Changes are far too numerous to enumerate; chief changes include
        that mappers are now produced from "configurations" built outside
        of those mappers, using a revised configuration API.    Events are
        available for constructing new events at the time a "configuration"
        is performed.   The internals have been streamlined both
        in terms of code organization and naming, and numerous long-standing
        feature requests have been implemented, including the ability to store
        a type object in a mapping against a type, greater control over
        the behavior of relationship loaders, control over the name used
        to identify tables in SQL, greater control over column accessor
        assignment, and significantly more.

        Major features supported by the new implementation include:

        - per-mapper session state initialization
        - easy single-inheritance mapping (ORM’s core trait)
        - flexible "configurations" supported by a new "MapperConfig"
          system
        - more granular control over loader query generation
        - a new "configuration events" system.
        - mapping to and from a type object itself, without an instance


    .. change::
        :tags: feature, orm
        :tickets: 2812

        Polymorphic loading is fully compatible with multiple table inheritance
        two levels deep.  Previously a single SQL statement could not be emitted
        when a subclass of the target class itself inherited from a subclass
        of the original class.

        .. seealso::

              :ref:`querying_with_polymorphic` 


    .. change::
        :tags: feature, orm
        :tickets: 1444

        Added a new hook,  :meth:`.SessionEvents.populate_existing` , to allow
        an existing row in the   :class:`.Session` 's identity map to be refreshed
        from the database so that it can present the current state of the
        database, bypassing any locally modified state.

    .. change::
        :tags: feature, orm
        :tickets: 1369

        Added event hooks for per-attribute instrumentation of transaction
        events, allowing more granular control over when an attribute is
        to be expired or flushed.

    .. change::
        :tags: feature, orm
        :tickets: 2021, 2356

        A new parameter ``raiseerr`` is now available within the ORM's
         :meth:`_session.Session.begin_nested`  method.  This parameter controls
        whether or not the `rollback` method is called automatically upon
        a rollback at the end of the block.

    .. change::
        :tags: feature, orm
        :tickets: 157

        A new flag has been added to the ORM’s
         :meth:`_orm.Registry.map_imperatively` 
        method to specify that the class be explicitly excluded from
        querying, rendering it effectively the same as a non-mapped class.

    .. change::
        :tags: feature, orm
        :tickets: 1781

        Added  :meth:`_orm.Query.join()`  method, which
        takes a target attribute name or column argument name - in conjunction
        with a new optional `from_joinpoint` parameter - to indicate that a
        join should begin from an alternate point.  This allows for more
        succinct and simpler joins to be specified.

    .. change::
        :tags: feature, orm
        :tickets: 1811

        Schema-level attributes and entities (tables, constraints,
        sequences) can now have their name format adjusted using a
        new  :paramref:`_schema.MetaData.naming_convention`  parameter.
        A "underscore" naming convention, as well as a "foreign key target"
        naming convention, are available in the default implementation.

    .. change::
        :tags: feature, orm
        :tickets: 219, 1966, 2171, 2619

        The legacy   :func:`_orm.reconstruct_instance`  function, used internally by
        the ORM since the beginning of time, has been refactored and made
        public.  This function provides basic object state transliteration
        services, in which a dictionary of simple attribute names and values
        is converted to an object state.  The function is also extensible
        via the  :paramref:`_orm.MapperExtension.reconstruct_instance` 
        method, which allows post-processing of object state as it is being
        built up.

    .. change::
        :tags: feature, orm
        :tickets: 1810

        The "class_" argument to the  :meth:`_orm.mapper`  function has been
        deprecated in favor of the more identifiable "class_".

    .. change::
        :tags: enhancement, sql
        :tickets: 2578

        In Oracle dialect, BOOLEAN types are now rendered as NUMBER(1,0),
        which is the most commonly utilized method for emulating boolean
        types in Oracle.

    .. change::
        :tags: feature, orm
        :tickets: 2768

        Added  :meth:`_orm.relationship.cascading_iterator` , a
        specialization of the standard Python ``iterator`` that iterates
        through an entire hierarchical tree of objects, fully expanding
        sub-relationships based on the current   :class:`_orm.Session`  state.

    .. change::
        :tags: feature, orm
        :tickets: 2812

        Polymorphic query strategies now supported on a per-mapper basis;
        strategies can be declared via the existing ``polymorphic_on`` attribute
        together with ``with_polymorphic()`` or directly on the  :meth:`_orm.mapper` 
        using a new  :paramref:`_orm.mapper.polymorphic_identity`  parameter.

        .. seealso::

              :ref:`mapping_inheritance` 


    .. change::
        :tags: feature, orm
        :tickets: 2606

        Added   :func:`_orm.with_polymorphic`  construct which allows for
        more concise usage of polymorphic loading.   The construct can be embedded
        directly in a query, or applied to a   :class:`_orm.relationship` .

    .. change::
        :tags: feature, orm
        :tickets: 2722

        Added support for using subset of columns from an inherited table
        in a   :class:`_orm.relationship` , by specifying the list of columns directly.

        .. seealso::

              :ref:`inheritance_relationship_supported_column_subset` 


    .. change::
        :tags: feature, postgresql
        :tickets: 2646

        Postgresql generate_series() is now available as an element in a SQL expression via the
        new   :class:`_postgresql.generate_series`  class.

    .. change::
        :tags: enhancement, sql

        Added the ability to generate "ON CONFLICT" clauses via the
         :meth:`_expression.Insert.on_conflict_do_update` ,  :meth:` _expression.Insert.on_conflict_do_nothing` 
        and related constructs.  A  :meth:`_.UpdateStatement.on_conflict_do_update` ,
        class, based off of   :class:`_expression.Insert.on_conflict_do_update` , is also
        now available.

        .. seealso::

              :ref:`postgresql_on_conflict` 


    .. change::
        :tags: feature, postgresql
        :tickets: 2523

        Added  :meth:`_ddl.SchemaDropper.remove_from_metadata` , and
        related methods for triggers, sequences, and views.

    .. change::
        :tags: feature, sql
        :tickets: 2604

        Added   :class:`_expression.Insert`  support for per-row default value generation
        using a dictionary-based convention or the new   :class:`_expression.Insert.frozenset_defaults()` 
        method.

    .. change::
        :tags: enhancement, postgresql
        :tickets: 2486

        Added support for   :class:`postgresql.ENUM`  constructs to be rendered
        inline within expressions, allowing them to be inserted into "SELECT"
        lists and expressions, or used within SQL constructs such as text().

    .. change::
        :tags: enhancement, postgresql
        :tickets: 2669

        Added support for Postgresql partial indexes on columns, via the
         :meth:`_postgresql.Index.where`  method.

    .. change::
        :tags: feature, postgresql
        :tickets: 2596

        Added support for Postgresql SP-GiST indexes.

    .. change::
        :tags: feature, postgresql
        :tickets: 2597

        Added support for Postgresql multi-column "INCLUDE" indexes, as
        well as "fillfactor" and "fastupdate" parameters via the
         :meth:`_postgresql.Index.include` ,  :meth:` _postgresql.Index.fillfactor` ,
        and  :meth:`_postgresql.Index.fastupdate`  methods.

    .. change::
        :tags: feature, pymysql
        :tickets: 2539

        With the PyMySQL driver, if a profile filename is passed using the
        "read_default_file" flag and the filename does not contain either
        an absolute path or a tilde expansion, the lookup process will be
        relative.

    .. change::
        :tags: enhancement, sql
        :tickets: 2537

        Rows inserted or updated through the "VALUES" clause (i.e. "INSERT INTO
        t VALUES (1, 2, 3)") will construct the required bound parameters by
        default, avoiding the need for a typecasting set of "bindparam()"
        constructs.

    .. change::
        :tags: feature, mysql
        :tickets: 2349

        Added support for the "ON DUPLICATE KEY UPDATE" MySQL syntax to
        the   :class:`_expression.Insert`  construct, so that this SQL syntax can be
        produced automatically when using  :meth:`_engine.Engine.execute`  or   :class:` _orm.session.Session.commit` .

        .. seealso::

              :ref:`mysql_on_duplicate_key_update` 


    .. change::
        :tags: feature, sql
        :tickets: 2176

        Added  :meth:`_expression.Update.prefix_with()`  method,
        which allows prefixing an UPDATE statement via a "prefix" string
        (e.g. "UPDATE [prefix] mytable SET column=value").

    .. change::
        :tags: feature, sql
        :tickets: 2076

        Added support for SQLite "partial indexes" with the new
         :meth:`_sqlite.Index.where`  method.

    .. change::
        :tags: enhancement, sql

        Added the  :meth:`_expression.TextClause.cte`  method to allow a CTE to be
        associated with an ad-hoc textual SQL expression.

    .. change::
        :tags: enhancement, sql
        :tickets: 1266

        A new version of the SQLite dialect that provides significant performance
        improvements to the speed at which SQLite returns result rows, up to 2x or
        more improvement over previous versions.

    .. change::
        :tags: enhancement, sql
        :tickets: 2605

        Added   :class:`_expression.values`  as a shorthand to the VALUES
        clause in INSERT.

    .. change::
        :tags: feature, sql
        :tickets: 2080

        Added a new  :meth:`_expression.Insert.return_defaults`  method; for
        Postgresql, this emits a RETURNING clause that returns the latest values
        of all columns inserted or updated, in effect returning the new primary
        key value for an inserted row.

    .. change::
        :tags: bug, sqlite
        :tickets: 2672

        Fixed a long-standing bug in the SQLite dialect that would cause cyclic
        foreign key constraints to not be applied.

    .. change::
        :tags: feature, sql
        :tickets: 869

        Added support for natural FULL and RIGHT joins and FULL OUTER
        JOINs within the  :meth:`_sql.Select.join`  method.

    .. change::
        :tags: enhancement, sql
        :tickets: 1926

        The PyODBC dialect now includes the ODBC version in the "odbc_connect"
        string.

        .. seealso::

              :ref:`dialects_pyodbc` 


    .. change::
        :tags: feature, sql
        :tickets: 2321

        The "returning" parameter is now supported by the MySQL and MariaDB dialects.

    .. change::
        :tags: bug, oracle
        :tickets: 297

        The  :meth:`_oracle.OracleDialect.connect`  method now sets
        the ORA_SDTZ environment variable, providing Oracle servers
        with the correct session time zone.

    .. change::
        :tags: feature, postgresql
        :tickets: 1865, 1882

        Added support for foreign key constraints on partitioned tables via
        the new  :paramref:`_postgresql.ForeignKeyConstraint.constrained_columns` 
        parameter.  Also added support for "DEFERRABLE" and "INITIALLY IMMEDIATE"
        constraints using the new  :paramref:`_postgresql.ForeignKeyConstraint.deferrable` 
        and  :paramref:`_postgresql.ForeignKeyConstraint.initially`  parameters.

    .. change::
        :tags: feature, postgresql
        :tickets: 2070

        The ``inline_literal()`` construct and ``bindparam()``
        can now handle PostgreSQL's array types, including nested arrays.

    .. change::
        :tags: feature, sql
        :tickets: 1501

        Multi-table UPDATEs can now make use of a "ORDER BY" clause via the
        new  :meth:`_expression.Update.order_by()`  method.

    .. change::
        :tags: feature, sql
        :tickets: 2462

        Added a new module-level function   :func:`_sql.column`  which provides
        a convenient shorthand for   :class:`_expression.ColumnElement`  generation.

    .. change::
        :tags: feature, sql
        :tickets: 1867

        The   :class:`_sql.Join`  construct can now generate "CROSS JOINs" by
        using the  :attr:`_sql.Join.isouter`  flag or using the string 'CROSS JOIN'
        directly.

    .. change::
        :tags: enhancement, sql
        :tickets: 2538

        MySQL-style function invocation parentheses are now canonicalized.

    .. change::
        :tags: feature, sql
        :tickets: 2085

        Added support for the "pragma" construct in SQLite via the new
         :meth:`_sqlite.SQLiteConnection.execution_options`  method.

        .. seealso::

              :ref:`sqlite_pragma` 


    .. change::
        :tags: feature, sql
        :tickets: 2042

        All "core" constructs that accept a textual SQL construct for its name
        attribute can now receive a textual UNION construct through the
        similar "unnion()" method, which will join two standalone selectables.

        .. seealso::

              :ref:`coretutorial_unions` 


    .. change::
        :tags: feature, sql
        :tickets: 2494

        All "core" constructs that work with a connection's schema
        can now optionally receive a schema name with the incoming
        bound parameter, using a new "named" bind parameter syntax.

        .. seealso::

              :ref:`sql_expression_names_bindparam` 


    .. change::
        :tags: bug, postgresql
        :tickets: 2474, 2473

        The   :class:`_postgresql.COMPOSITE`  type is now more fully supported within the
        ORM, allowing values passed to/from the DB to be treated as if they were
        a row of columns when embedded within a row of columns or nested structure.

        Additionally, the Postgresql cipher extension is now detected and its
        type added to the type map.

    .. change::
        :tags: bug, sql
        :tickets: 1952

        Fixed performance issue when using result set buffering in some DBAPIs
        such as sqlite3 and MySQLdb, where many hundreds of thousands of
        rows are fetched at once.

    .. change::
        :tags: feature, sql
        :tickets: 1106

        A new SQL filter system is now in place which can significantly reduce
        the number of queries required for some common types of queries.
        The feature allows individual methods and SELECT statements to include
        a filtering vendor-specific WHERE clause which allows related entities
        to be joined.  This provides a way to cut down on confusing
        ORM-generated queries that join in a large number of tables.

    .. change::
        :tags: bug, sql
        :tickets: 2066

        Corrected a bug in statement caching which compares bound values
        to determine cacheable uniqueness.  Previously, if bound values from
        a prior statemenet and a new statement are equal but are composed of
        different Python-typed objects, such as datetime.date() versus
        datetime.datetime(), the values would not be correctly cached.

        This caching strategy is now more specifically tied to the SQLA integer/float/string type
        equivalents, as well as handling for datetime.time() and datetime.date()
        objects.


    .. change::
        :tags: feature, sql
        :tickets: 2237

        A new  :meth:`_expression.Select.literal_column`  method has been added to SQLAlchemy
        which returns a column expression object which does not support typical
        "bind parameter" behavior, but is useful in SELECT statements where columns
        need to be produced that represent specific fixed expressions.


    .. change::
        :tags: feature, sql
        :tickets: 2277

        The syntax "table AS alias(column1, column2, ...)" can now be used
        to specify columns and a table alias name simultaneously.

    .. change::
        :tags: feature, sql
        :tickets: 1396

        A new   :func:`_expression.bound_expression`  construct has been added to
        provide a way to specify the type of a bound expression as well as an
        optional function or callable that will be used to convert
        Python values passed to the select or statement parameter.

    .. change::
        :tags: enhancement, sql
        :tickets: 2312

        Bound parameters can now be coerced to one of several Python types
        upon execution by passing a dictionary to the  :meth:`_engine.Connection.execute` 
        method via the "type\_coerce" key.










.. _changelog_0_9_1:

Changelog for version 0.9.1
===========================

.. note:: This release contains a number of bugfixes only.

Bug Fixes
---------

.. _change_0_9_1_bug_01:

- schema
  - Fixed a bug involving the new flattened JOIN structures which are used with 
      :func:`_orm.joinedload()` .

.. _change_0_9_1_bug_02:

- engine
  - Fixed an issue where the C extensions in Py3K are using the wrong API to specify 
    the top-level module function, which breaks in Python 3.4b2.
    
.. _change_0_9_1_bug_03:

- orm
  - Fixed bug where using a  :attr:`.Session.info`  attribute would fail if the 
    ``.info`` argument were only passed to the   :class:`.sessionmaker`  creation 
    call but not to the object itself.
    
.. _change_0_9_1_bug_04:

- orm
  - Fixed bug where we don't check the given name against the correct string class 
    when setting up a backref based on a name.
    
.. _change_0_9_1_bug_05:

- sql
  - The precedence rules for the  :meth:`.ColumnOperators.collate`  operator have been 
    modified, such that the COLLATE operator is now of lower precedence than the 
    comparison operators.
    
.. _change_0_9_1_bug_06:

- sqlalchemy.ext.automap
  - Fixed an extremely unlikely memory issue where when using   :class:`.DeferredReflection`  
    to define classes pending for reflection, if some subset of those classes were 
    discarded before the  :meth:`.DeferredReflection.prepare`  method were called to 
    reflect and map the class, a strong reference to the class would remain held 
    within the declarative internals.
    
.. _change_0_9_1_bug_07:

- orm
  - Fixed bug where we apparently still create an implicit alias when saying 
    query(B).join(B.cs), where "C" is a joined inh class.
    
.. _change_0_9_1_bug_08:

- orm
  - The ``viewonly`` flag on   :func:`_orm.relationship`  will now prevent attribute history 
    from being written on behalf of the target attribute.
    
.. _change_0_9_1_bug_09:

- orm
  - Added support for new  :attr:`.Session.info`  attribute to   :class:` .scoped_session` .

.. _change_0_9_1_bug_10:

- orm
  - Fixed bug where usage of new   :class:`.Bundle`  object would cause the  :attr:` _query.Query.column_descriptions`  
    attribute to fail.

.. _change_0_9_1_bug_11:

- examples
  - Fixed bug which prevented history_meta recipe from working with joined inheritance 
    schemes more than one level deep.

.. _change_0_9_1_bug_12:

- sql, postgresql, mysql
  - Fixed a bug that was preventing reflection/inspection of foreign key options.


Feature Enhancements
--------------------

This release does not contain any feature enhancements.修复和测试从反射中解析MySQL外键选项的问题；这与  :ticket:`2183`  中完成对外键选项（如 ON UPDATE/ON DELETE cascade 等）的反射支持相补充。

.. change::
    :tags: bug, orm
    :tickets: 2787

    当与标量列映射的属性一起使用时，  :func:`.attributes.get_history()`  现在会遵循传递给它的“passive”标志；
    因为默认值为 ``PASSIVE_OFF``，所以如果值不存在，该函数将默认查询数据库。
    这是与0.8中的行为不同的行为更改。

    .. seealso::

          :ref:`change_2787` 

.. change::
    :tags: feature, orm
    :tickets: 2787

    添加新方法  :meth:`.AttributeState.load_history` ，与  :attr:` .AttributeState.history`  类似，
    但也触发加载器可调用函数。

    .. seealso::

          :ref:`change_2787` 


.. change::
    :tags: feature, sql
    :tickets: 2850

    当在类型化的表达式中使用没有指定类型的   :func:`.bindparam`  构造时，会复制该构造当时的状态，并将新副本分配给与之进行比较的列的实际类型。
    以前，这种逻辑会就地在给定的   :func:`.bindparam`  上进行。
    此外，在编译阶段的  :meth:`.ValuesBase.values`  用于   :class:` _expression.Insert`  或   :class:`_expression.Update`  构造中，现在也会发生类似的过程，这些构造在   :func:` .bindparam`  构造中使用。

    这两个都是微妙的行为更改，可能会影响一些用法。

    .. seealso::

          :ref:`migration_2850` 

.. change::
    :tags: feature, sql
    :tickets: 2804， 2823， 2734

    对特殊符号的表达式处理进行了修订，特别是与连接相关的符号，例如
    ``None``   :func:`_expression.null`    :func:` _expression.true` 
      :func:`_expression.false` ，
    包括在连锁的且 / 或表达式中呈现 NULL 的一致性，
    包含在布尔常量和表达式的后端的形式为“1”或“0”。
    对于没有“true”/“false”常量的后端系统，这些常量和表达式的呈现方式。

    .. seealso::

          :ref:`migration_2804` 

.. change::
    :tags: feature, sql
    :tickets: 2838

    类型系统现在处理呈现“文字绑定”值的任务，例如通常将绑定参数绑定为字符串，
    但是由于上下文必须在 DDL 结构内呈现，例如 CHECK 约束和索引（请注意，“文字绑定”值作为  :ticket:`2742`  的 DDL 的一部分而被使用）。
    一个新方法  :meth:`.TypeEngine.literal_processor`  作为基础，
    并添加了  :meth:`.TypeDecorator.process_literal_param`  以允许包装本地文字呈现方法。

    .. seealso::

          :ref:`change_2838` 

.. change::
    :tags: feature, sql
    :tickets: 2716

    现在  :meth:`_schema.Table.tometadata`  方法会复制该结构中所有   :class:` .SchemaItem`  对象的  :attr:`.SchemaItem.info`  字典，
    包括列，约束，外键等。由于这些字典是副本，因此它们独立于原始字典。
    以前，此操作仅传输了   :class:`_schema.Column`  的 ` `.info`` 字典，并且它仅被链接而非复制。

.. change::
    :tags: feature, postgresql
    :tickets: 2840

    使用 PostgreSQL 版本 9.2 或更高版本的服务器版本检测时，
    当在主键自动递增列上使用   :class:`.SmallInteger`  类型时，添加了对“SMALLSERIAL”的渲染支持。

.. change::
    :tags: feature, mysql
    :tickets: 2817

    MySQL 的   :class:`.mysql.SET`  类型现在具有与   :class:` .mysql.ENUM`  相同的自动引号行为。
    在设置值时不需要引号，但是将自动检测到存在的引号，并发出警告。
    这还有助于 Alembic，其中 SET 类型不使用引用呈现。

.. change::
    :tags: feature, sql

    现在   :class:`_schema.Column`  的 ` `default`` 参数接受类或对象方法作为参数，除了作为独立函数外；
    它会正确检测是否接受了“context”参数或未接受它。

.. change::
    :tags: bug, sql
    :tickets: 2835

    在“attach”事件被调用之前，将“name”属性设置到   :class:`.Index`  上，
    以便可以使用附加事件基于父表或列动态为索引生成名称。

.. change::
    :tags: bug, engine
    :tickets: 2748

    已经改进了  :meth:`.Dialect.reflecttable`  的方法签名，该签名在现有情况下通常由   :class:` .DefaultDialect`  提供，
    带有 ``include_columns`` 和 ``exclude_columns`` 参数，而没有任何关键字选项，从而减少了歧义；先前丢失了 ``exclude_columns``。

.. change::
    :tags: bug, sql
    :tickets: 2831

      :class:`_schema.ForeignKey`  对象中的错误 kw arg “schema”已被删除。
    这是一个意外的提交，未生效 ; 在使用此 kw arg 时，会在0.8.3中引发警告。

.. change::
    :tags: feature, orm
    :tickets: 1418

    添加了一个新的加载选项   :func:`_orm.load_only` 。
    这允许指定一系列列名以仅加载这些属性，推迟其余部分。

.. change::
    :tags: feature, orm
    :tickets: 1418

    接口函数的系统已完全重新架构为建立在更全面的基础上，即   :class:`_orm.Load`  对象。
    此基为通用加载选项（如   :func:`_orm.joinedload` ，  :func:` .defer`  等）提供了一种连接样式，
    用于指定路径下的选项，例如 “joinedload(“foo”).subqueryload(“bar”)”。
    新系统取代了点分隔路径名称，选项中的多个属性以及使用 ``_all()`` 选项的用法。

    .. seealso::

          :ref:`feature_1418` 

.. change::
    :tags: feature, orm
    :tickets: 2824

      :func:`.composite`  构造现在在基于列的   :class:` _query.Query`  中维护返回对象，
    而不是展开为单独的列。这在内部使用了新的   :class:`.Bundle`  功能。
    此行为与旧版不兼容；要选择展开的复合列，可使用 ``MyClass.some_composite.clauses``。

    .. seealso::

          :ref:`migration_2824` 

.. change::
    :tags: feature, orm
    :tickets: 2824

    添加了一个新的构造   :class:`.Bundle` ，它允许将列表的组合指定给   :class:` _query.Query`  构造。
    默认情况下，这些列的组合作为元组返回。  :class:`.Bundle`  的行为可以被覆盖，以提供任何类型的结果处理。
    此外，在列导向的   :class:`_query.Query`  中使用复合属性时，现在将内置   :class:` .Bundle`  功能嵌入到其中。

    .. seealso::

          :ref:`change_2824` 

          :ref:`migration_2824` 

.. change::
    :tags: bug, sql
    :tickets: 2812

    “引用”标识符的处理方式进行了大幅更改，不再依赖于传递各种 ``quote=True`` 标志，
    而是将这些标志转换为富字符串对象，这些对象包含在常用模式构造中传递的引号信息，例如   :class:`_schema.Table` 、
      :class:`_schema.Column`  等。这解决了许多方法无法正确遵守“引用”标志的问题，
    例如  :meth:`_engine.Engine.has_table`  和相关方法。  :class:` .quoted_name`  对象是一个字符串子类，
    如果需要，也可以显式使用该对象；该对象将保持传递的引号首选项，
    并且还将绕过面向大小写符号的方言所执行的“名称规范化”。例如 Oracle、Firebird 和 DB2。
    结果是，“大写”后端现在可以使用强制引用名称，例如小写引用名称和新保留字。

    .. seealso::

          :ref:`change_2812` 

.. change::
    :tags: feature, orm
    :tickets: 2793

    ``Mapper`` 的 ``version_id_generator`` 参数现在可以指定依赖于服务器生成的版本标识符，
    使用触发器或其他提供的数据库版本控制功能，或通过设置 ``version_id_generator=False`` 的可选编程值。
    当使用由服务器生成的版本标识符时，ORM 将在 RETURNING 可用时立即使用该值加载新版本值，
    否则它将发出第二个 SELECT。

.. change::
    :tags: feature, orm
    :tickets: 2793

    现在   :class:`_orm.Mapper`  的 ` `eager_defaults`` 标志允许使用内联 RETURNING 子句而不是第二个 SELECT 语句来获取新生成的默认值，
    对于支持 RETURNING 的后端。

.. change::
    :tags: feature, core
    :tickets: 2793

    添加了一个  :meth:`.UpdateBase.returning`  的新变体，称为  :meth:` .ValuesBase.return_defaults` ；
    这允许向语句的 RETURNING 子句添加任意列，而不会影响编译器的通常“隐式返回”功能，该功能用于有效地获取新生成的主键值。
    对于支持的后端，所有获取的值的字典存在于  :attr:`_engine.ResultProxy.returned_defaults`  中。

.. change::
    :tags: bug, mysql

    改进了 cymysql 驱动程序的支持，支持版本0.6.5，由Hajime Nakagami提供。

.. change::
    :tags: general

    对于核心模块以及 ORM 模块的某些方面，有大量包重构重新组织了导入结构。
    特别是，``sqlalchemy.sql`` 已分为比以前更多的模块，因此 ``sqlalchemy.sql.expression`` 的大小现在缩小了。
    努力集中在大大减少导入周期。
    此外，API 函数在 ``sqlalchemy.sql.expression`` 和 ``sqlalchemy.orm`` 中的系统已重新组织，
    以消除函数与它们产生的对象之间的文档中的冗余。

.. change::
    :tags: orm, feature, orm

    添加了一个新属性  :attr:`.Session.info`  给   :class:` .Session` ；
    这是一个字典，应用程序可以将任意数据存储在   :class:`.Session`  中。
     :attr:`.Session.info`  的内容还可以使用   :class:` .Session`  或   :class:`.sessionmaker`  的“info”参数来初始化。


.. change::
    :tags: feature, general, py3k
    :tickets: 2161

      :func:`~sqlalchemy.sql.expression.label`  构造现在在 ` `ORDER BY`` 子句中会仅呈现其名称，而不是重写整个表达式的名称，如果也在 select 的列子句中引用了该标签。
    这给数据库更好的机会在两个不同的上下文中评估同一表达式。

    .. seealso::

          :ref:`migration_1068` 

.. change::
    :tags: feature, engine
    :tickets: 2770

      :class:`_events.ConnectionEvents`  增加了新事件：

    *  :meth:`_events.ConnectionEvents.engine_connect` 
    *  :meth:`_events.ConnectionEvents.set_connection_execution_options` 
    *  :meth:`_events.ConnectionEvents.set_engine_execution_options` 

.. change::
    :tags: bug, sql
    :tickets: 1765

    将   :class:`_schema.ForeignKey`  对象解析为其目标   :class:` _schema.Column`  的方式已进行了重新处理，
    以尽可能即时地进行关联到具有相同   :class:`_schema.MetaData`  的对象的目标   :class:` _schema.Column` ，
    而不是等待第一次构建联接或类似操作。这连同其他一些改进使得更早地检测到某些外键配置问题。
    此外，对于通过   :class:`_schema.ForeignKey`  引用另一个列的任何   :class:` _schema.Column` ，
    应该现在可以可靠地将类型设置为 ``None`` - 该类型将在关联的那一侧列关联时从目标列复制，
    现在也支持复合外键。

    .. seealso::

          :ref:`migration_1765` 

.. change::
    :tags: feature, sql
    :tickets: 2744, 2734

    为   :class:`.TypeDecorator`  添加了一种名为  :attr:` .TypeDecorator.coerce_to_is_types`  的新属性，
    以便更轻松地控制如何进行使用“==”或“！=”到 None 和布尔类型的比较，
    生成“IS”表达式或带有绑定参数的普通等式表达式。

.. change::
    :tags: feature, pool
    :tickets: 2752

    添加池日志记录，“rollback-on-return”和较少使用的“commit-on-return”功能。这是使用池“debug”日志记录启用的。

.. change::
    :tags: bug, orm, associationproxy
    :tickets: 2751

    对于使用标量值进行比较的 ==、！= 比较符，修复了一个晦涩的错误，
    当跨越一对多关系连接/联接加载到具有特定鉴别器值的单表继承子类时，
    会获取到错误的结果，因为将返回“secondary”行。
    现在，在所有ORM连接的多对多关系上，将右侧的“secondary”和表都内部连接在括号中，
    以便左->右连接可以准确地进行过滤。这个变化的可能是：= Cls.scalar == None”比较将返回“Cls 安装”存在于和“Cls 关联”。关联.scalar 是无值还是 NULL，之前这样处理也是错误的。

    修改“Cls.标量！='somevalue'”的情况，以更像SQL比较；
    只返回相关的行，即当“Cls.相关”存在且“Associated.scalar”非 NULL 且不等于 ``'somevalue'`` 时。
    以前，这将是一个简单的“NOT EXISTS”。

    还添加了一个特殊用例，在不带参数调用 “Cls.scalar.has()”时使用，当``Cls.scalar``是基于列的值时，
    这将返回"Cls.associated"是否存在任何行，而不管“Cls.associated.scalar”是否为 NULL。


    .. seealso::

          :ref:`migration_2751` 

.. change::
    :tags: feature, orm
    :tickets: 2587

    关于 ORM 如何构建联接，其中右侧本身是一个连接或左外联接的方式进行了重大变更。
    现在 ORM 已配置为允许简单嵌套形式的联接，例如 ``a JOIN (b JOIN c ON b.id=c.id) ON a.id=b.id``，而不是将右侧强制转换为 “SELECT” 子查询。
    这应该允许大多数后端获得显着的性能提升，特别是MySQL。
    长达多年一直阻碍此更改的唯一数据库后端—— SQLite，现已通过将“SELECT” 子查询的生成从 ORM 移动到 SQL 编译器来解决。
    因此，SQLite 上的右侧嵌套联接仍将最终呈现为“SELECT”，而所有其他后端则不再受此解决方案的影响。

    作为此更改的一部分，已将新参数 ``flat=True`` 添加到   :func:`_orm.aliased` 、  :meth:` _expression.Join.alias`   和
      :func:`_orm.with_polymorphic`  函数中，该参数允许生成连接的“别名”，该别名为连接内的每个组件表应用匿名别名，
    而不是产生子查询。

    .. seealso::

          :ref:`feature_joins_09` 

.. change::
    :tags: bug, orm
    :tickets: 2369

    修复了一个晦涩的错误，在跨越多对多关系连接或联接跨越单个表继承子类时，将具有特定鉴别器值错误地获取错误的结果，由于“secondary”行。
    现在，在所有ORM连接的多对多关系上，将“secondary”和右侧表内部连接在括号中，以便左->右连接可以准确进行过滤。
    此更改是通过终于解决  :ticket:`2587`  中概述的右嵌套连接问题而实现的。

    .. seealso::

          :ref:`feature_joins_09` 

.. change::
    :tags: bug, mssql, pyodbc
    :tickets: 2355

    修复了 MSSQL 的 Pyton 3 + Pyodbc 中的错误，包括正确传递语句。

.. change::
    :tags: feature, sql
    :tickets: 1068

    如果在选择的列子句中也引用该标签，则   :func:`~sqlalchemy.sql.expression.label`  构造现在在 ` `ORDER BY`` 子句中仅将其名称呈现为名称，
    而不是重写完整表达式。这使数据库更有机会优化不同上下文中同一表达式的评估。

    .. seealso::

          :ref:`migration_1068` 

.. change::
    :tags: feature, firebird
    :tickets: 2504

    当未使用方言限定符时，即，即 ``firebird://``，
    ``fdb`` 方言现在是默认方言，因为 Firebird 项目将 ``fdb`` 发布为其公认的 Python 驱动程序。

.. change::
    :tags: feature, general, py3k
    :tickets: 2671

    代码库现在为 Python 2 和 3 进行“原地”处理，不需要运行 2to3。
    实现的兼容性是 Python 2.6 及更高版本。

.. change::
    :tags: feature, oracle, py3k

    使用 cx_oracle 时，Oracle 单元测试现在完全通过 Python 3。 

.. change::
    :tags: bug, orm
    :tickets: 2736

     :meth:`_query.Query.select_from`  方法的“自动别名”行为已关闭。特定行为现在可通过一个新方法  :meth:` _query.Query.select_entity_from`  实现。
    此处的自动别名行为从未有过很好的记录，并且通常不是所需的，
    因为  :meth:`_query.Query.select_from`  变得更加定位于控制如何呈现 JOIN。  :meth:` _query.Query.select_entity_from`   
    也将在 0.8 中提供，以便依赖自动别名的应用程序可以将其应用程序转向使用此方法。

    .. seealso::

          :ref:`migration_2736` 