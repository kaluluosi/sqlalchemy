========
术语表
========

.. glossary::
    :sorted:

    1.x style
    2.0 style
    1.x-style
    2.0-style
        这些术语是SQLAlchemy 1.4中的新术语，指的是SQLAlchemy 1.4->2.0转换计划，该计划在 :ref:`migration_20_toplevel`中描述。
        “1.x样式”是指在1.x系列和早期（例如1.3,1.2等）中记录了API的使用方式，“2.0样式”是指API在版本2.0中的外观。
        版本1.4在“转换模式”下实现了几乎所有2.0的API，而版本2.0仍然维护传统的:class:`_orm.Query`对象，以允许遗留代码基本上与2.0兼容。

        .. seealso::

            :ref:`migration_20_toplevel`

    sentinel
    插入的标志
        这是一种SQLAlchemy特有的术语，用于表示可以用于大量插入操作的:class:`_schema.Column`，以跟踪使用RETURNING或类似方法返回的行与传递的行。当
        :term:`insertmanyvalues`功能一次性针对多个行进行优化的INSERT..RETURNING语句并仍然能够保证返回的行的顺序与输入数据匹配时，
        此类列配置是必需的。

        对于典型用例，SQLAlchemy SQL编译器可以自动使用替代整数主键列作为“插入的标志码”，并且不需要任何用户配置。对于其他类型的
        服务器生成的主键值的不常见情况，可以在:term:`table metadata`中显式地配置“插入标志”列，以便优化一次性插入多行的插入语句。

        .. seealso::

            :ref:`engine_insertmanyvalues_returning_order` - 在 :ref:`engine_insertmanyvalues`部分中

    insertmanyvalues
        这是指一种SQLAlchemy特有的功能，允许INSERT语句在单个语句中发出数千个新行，同时允许使用RETURNING或类似的方法内联从语句返回的服务器生成的值以进行性能优化。该特性旨在对选择的后端透明地可用，
        但确实提供了一些配置选项。 有关此功能的完整说明，请参见本特性的部分 :ref:`engine_insertmanyvalues`。

        .. seealso::

            :ref:`engine_insertmanyvalues`

    mixin类
    mixin类别
        这是一种常见的面向对象模式，可由包含其他类使用的方法或属性而无需成为这些其他类的父类来完成。

        .. seealso::

            `Mixin（通过维基百科） <https://en.wikipedia.org/wiki/Mixin>`_

    reflection
    反射
    反映
        在SQLAlchemy中，此术语指查询数据库的模式目录以获取有关现有表、列、约束和其他构造的信息的功能。 SQLAlchemy包括
        可以为此信息提供原始数据的功能，以及可以自动从数据库模式目录构造Core / ORM可用的:class:`.Table`对象的功能。

        .. seealso::

            :ref:`metadata_reflection_toplevel` - 数据库反射的完整背景。

            :ref:`orm_declarative_reflected` - 有关将ORM映射与反射表格集成的背景。

    imperative
    声明性
        在SQLAlchemy ORM中，这些术语指将Python类映射到数据库表的两种不同样式的映射。

        .. seealso::

            :ref:`orm_declarative_mapping`

            :ref:`orm_imperative_mapping`

    facade
        一个充当复杂底层或结构化代码的前置接口的对象。

        .. seealso::

            `Facade pattern（通过维基百科） <https://en.wikipedia.org/wiki/Facade_pattern>`_

    关系
    关系代数
        由Edgar F. Codd开发的一种代数系统，用于对存储在关系数据库中的数据进行建模和查询。

        .. seealso::

            `关系代数（通过维基百科） <https://en.wikipedia.org/wiki/Relational_algebra>`_

    笛卡尔积
        给定两个集合A和B，笛卡尔积是所有有序对（a，b）的集合，其中a在A中，b在B中。

        在SQL数据库中，当我们在不建立一个表与另一个表之间的任何准则（直接或间接）的情况下从两个或多个表格（或其他子查询）选择时，
        就会发生笛卡尔积。如果我们同时从表A和表B查询，我们将得到A的每一行与B的第一行配对，然后是将A的每一行与B的第二行匹配，
        以此类推，直到A的每一行都与B的每一行配对。

        笛卡尔积会导致生成大量结果集并且很容易使客户端应用程序崩溃。

        .. seealso::

            `笛卡尔积（通过维基百科） <https://en.wikipedia.org/wiki/Cartesian_product>`_

    圈度复杂度
        基于程序源代码的可能路径数的代码复杂度措施。

        .. seealso::

            `圆度复杂度 <https://en.wikipedia.org/wiki/Cyclomatic_complexity>`_

    绑定参数
    绑定的参数
    参数绑定
        案是DBAPI数据库驱动程序中数据传输的主要方式。虽然操作基于SQL语句字符串，但数据值本身是分开传递的，其中驱动程序包含将安全处理这些字符串并将它们传递到
        发往后端的数据库服务器的逻辑，后端数据库服务器可以将它们格式化到SQL字符串本身中，或使用单独的协议将它们传递到数据库中。

        数据库驱动程序执行此操作的特定方式对调用程序员来说不应该要紧；关键是在外部，数据应始终作为单独的部分而不是作为SQL字符串本身的一部分进行传递。
        这对于具有足够安全防范逆向注入的安全性以及使驱动程序具有最佳性能都至关重要。

        .. seealso::

            `Prepared Statement <https://en.wikipedia.org/wiki/Prepared_statement>`_ - 在维基百科上

            `绑定参数 <https://use-the-index-luke.com/sql/where-clause/bind-parameters>`_ - Use The Index, Luke

            :ref:`tutorial_sending_parameters` - 在 :ref:`unified_tutorial`中

    选择的
        在SQLAlchemy中使用的一种术语，用于描述表示集合的SQL构造。 它在很大程度上类似于 :term:`relational algebra` 中的“关系”概念。
        在SQLAlchemy中，子类化 :class:`_expression.Selectable` 类的对象被视为可在使用SQLAlchemy Core时可用的“选择器”。 最常见的两个构造是
        :class:`_schema.Table` 和 :class:`_expression.Select` 语句。

    ORM注释
    注释
        术语“ORM-annotated”是指SQLAlchemy的一个内部方面，其中例如 :class:`_schema.Column` 的Core对象可以携带额外的运行时信息，
        用于标记其属于特定ORM映射。该术语不应与常见的“类型注释”短语混淆，后者是指用于静态类型的Python源代码“类型提示”，如 :pep:`484` 中介绍的。

        大多数SQLAlchemy的文档代码示例都使用“带注释的示例”或“未带注释的示例”进行格式化。这指的是示例是否 :pep:`484` 带注释，
        与SQLAlchemy的“ORM-标注”概念无关。

        在文档中出现“ORM-annotated”短语时，它是指Core SQL表达式对象，例如 :class:`.Table`，: class:`.Column` 和 :class:`.Select` 对象，
        它们起源于或引用间接与一个或多个ORM映射相关联的子元素的对象，并因此在传递给ORM方法（例如 :meth:`_orm.Session.execute`）
        时将具有ORM特定的解释和/或行为。例如，当我们从ORM映射构造一个:class:`.Select` 对象时，例如在 :ref:`ORM Tutorial <tutorial_declaring_mapped_classes>`中所示的
        ``User`` 类::

            >>> stmt = select(User)

        以上 :class:`.Select` 的内部状态是指 ``User`` 映射到的:class:`.Table`。实际上，“User”
        类本身没有立即引用。这是 :class:`.Select` 对象保持与Core级别进程兼容的方式（请注意，:class:`.Select` 的 ``._raw_columns`` 成员是私有的，
        结束用户代码不应访问它）::

            >>> stmt._raw_columns
            [Table('user_account', MetaData(), Column('id', Integer(), ...)]

        但是，当我们的 :class:`.Select` 传递给ORM :class:`.Session` 时，
        与该对象间接关联的ORM实体将用于ORM上下文中解释此 :class:`.Select`。实际的“ORM注释”可以在另一个私有变量中看到 ``._annotations``:

          >>> stmt._raw_columns[0]._annotations
          immutabledict({
            'entity_namespace': <Mapper at 0x7f4dd8098c10; User>,
            'parententity': <Mapper at 0x7f4dd8098c10; User>,
            'parentmapper': <Mapper at 0x7f4dd8098c10; User>
          })

        因此，我们将 ``stmt`` 称为 **具有ORM注释的select()** 对象。它是一个 :class:`.Select` 语句，其中包含其他信息，
        当传递给 :meth:`_orm.Session.execute` 等ORM方法时，将导致它以ORM特定的方式进行解析。


    插件
    插件启用
    插件特定
        “插件启用”或“插件特定”通常表示在ORM上下文中使用某些函数或方法时其行为将如何不同。

        SQLAlchemy允许Core构造，如 :class:`_sql.Select` 对象，参与“插件”系统，该系统可以将其他功能和功能注入到默认情况下不存在的
        对象中。具体来说，主要的“插件”是“orm”插件，在这个插件系统中，SQLAlchemy ORM使用Core构造来组合和执行返回ORM结果的SQL查询。

        .. seealso::

            :ref:`migration_20_unify_select`

    crud
    CRUD
        一个简称，意思是“Create，Update，Delete”。 SQL中这个术语是指用于在数据库中创建、修改和删除数据的操作集，也称为 :term:`DML`，通常指
        “INSERT”，“UPDATE”和“DELETE”语句。

    executemany
        此术语指的是 :pep:`249` DBAPI规范的一部分，指针对数据库连接的多个参数集执行的单个SQL语句。特定的方法称为
        `cursor.executemany() <https://peps.python.org/pep-0249/#executemany>`_，与用于单个语句调用的 `cursor.execute() <https://peps.python.org/pep-0249/#execute>`_ 方法具有
        许多行为差异。 “executemany” 方法对传递的SQL语句执行多次，每次使用一个参数集。使用executemany的基本理由是改进性能，其中DBAPI可以使用多种技术，例如仅
        在执行之前准备该语句，或者以其他方式对最初的执行很多次的相同语句进行优化。

        SQLAlchemy通常在传递了参数字典列表的情况下自动使用 ``cursor.executemany()`` 方法，这表明SQL语句和处理过的参数集应该被传递到
        ``cursor.executemany()``，其中语句将被驱动程序个别地执行为每个参数字典。 ``cursor.executemany()`` 方法作为用
         已知所有DBAPI中的限制之一在使用时的一个关键限制是当该方法用于时 "INSERT..RETURNING"类似语句时标准（一个值得注意的例外是 cx_Oracle / OracleDB
         DBAPI）不会在每个INSERT执行中完成。例如，通常不能直接使用 ``cursor.executemany()`` 的多个参数数据，因为DBAPI通常不会将每个
         INSERT执行的单个行合并在一起。

        为了克服这个限制，从2.0系列开始，SQLAlchemy实现了另一种形式的“executemany”，称为 :ref:`engine_insertmanyvalues`。这个功能将使用
        ``cursor.execute()`` 执行INSERT语句，以便一次在多个参数集中进行，并在一次往返中执行相应的网络流量，从而产生与使用 ``cursor.executemany()`` 相同
        的效果，同时仍支持 RETURNING。

        .. seealso::

            :ref:`tutorial_multiple_parameters` - “executemany”的教程介绍

            :ref:`engine_insertmanyvalues` - SQLAlchemy功能，它允许将RETURNING与“executemany”一起使用

    过程化
    声明性
        在SQLAlchemy ORM中，这些术语指的是Python类与数据库表之间映射的两种不同样式。

        .. seealso::

            :ref:`orm_declarative_mapping`

            :ref:`orm_imperative_mapping`

    倍增
    倍增类别
        当将一个类与 :class:`_orm.Mapper` 类的实例相关联时，我们称该类为“映射”。此过程将该类与数据库表或其他 :term:`selectable` 构造相关联，
        以便可以使用 :class:`.Session` 持久化和加载它的实例。

        .. seealso::

            :ref:`orm_mapping_classes_toplevel`

    N加一问题
    N加一
        N加一问题是 :term:`lazy load` 模式的常见副作用，应用程序希望迭代结果集中每个对象的相关属性或集合，并且该属性或集合设置为通过该模式进行加载。
        净结果是发出一个SELECT语句来加载父对象的初始结果集。然后，当应用程序遍历每个成员时，对于每个成员都会发出一个其他的SELECT语句，
        以从数据库中加载其相关属性或集合。最终结果是，对于N个父对象的结果集，会发出N+1个SELECT语句。

        N加一问题使用 :term:`eager loading` 来减轻。

        .. seealso::

            :ref:`tutorial_orm_loader_strategies`

            :doc:`orm/queryguide/relationships`

    多态
    多态地
        指处理多个类型的功能。 在SQLAlchemy中，此术语通常应用于ORM映射的概念，根据结果集中的信息返回不同的子类，通常是通过检查标
        记在结果集中的特定列的值。

        SQLAlchemy中的多态加载意味着使用三种不同的方案之一或组合来映射层次结构的类；“joined”，“single”和“concrete”。 :ref:`inheritance_toplevel` 部分完整地描述了继承映射。

    方法链接生成式
    在SQLAlchemy文档中被称为“生成式”的“方法链接”是一种面向对象的技术，其中通过在对象上调用方法构建对象的状态。对象具有任意数量的方法，每个方法返回一个新对象（或在某些情况下相同的对象），并向对象添加其他状态。

    使用方法链接最多的两个SQLAlchemy对象是:class:`_expression.Select`对象和:class:`.orm.query.Query`对象。例如，可以通过调用:meth:`_expression.Select.where`和:meth:`_expression.Select.order_by`方法向:class:`_expression.Select`对象的WHERE子句分配两个表达式以及一个ORDER BY子句：

        stmt = (
            select(user.c.name)
            .where(user.c.id > 5)
            .where(user.c.name.like("e%"))
            .order_by(user.c.name)
        )

    上面的每个方法调用都返回原始的:class:`_expression.Select`对象的副本，并添加了其他限定符。

发布
发布版
已发布
    在SQLAlchemy上下文中，“已发布”一词是指结束使用特定数据库连接的过程。 SQLAlchemy支持连接池的使用，允许配置数据库连接的寿命。在使用池连接时，“关闭”它，即调用类似“connection.close()”的语句可能有以下效果：该连接被返回到现有池，或者它可能会导致实际关闭由该连接引用的底层TCP/IP连接-哪个取决于配置以及池的当前状态。因此，我们使用“已发布”这个术语，表示“在使用它们后，要做任何你计划做的有关连接的操作”。

    该术语有时会用于短语“释放事务资源”，以明确表示我们实际上“正在释放”连接所累积的任何事务状态。在大多数情况下，从表中选择，发出更新等操作会在该连接上获取:term：`孤立的`状态以及潜在的行或表锁定。此状态都是特定事务中本地的，并在我们发出回滚时释放。连接池的一个重要功能是，当我们将连接返回到池时，也会调用DBAPI的“connection.rollback()”方法，以便在准备再次使用连接时，它处于“干净”状态，没有对前一系列操作持有的引用。

    .. seealso::

        :ref:`pooling_toplevel`

DBAPI
PEP-249
    DBAPI是短语“Python数据库API规范”的缩写。这是Python中广泛使用的规范，用于为所有数据库连接包定义公共的使用模式。DBAPI是一个“低级”API，在Python应用程序中通常是使用的最低级别的系统，用于与数据库通信。 SQLAlchemy的:term:`dialect`系统是围绕DBAPI的操作构建的，为特定的数据库引擎和DBAPI服务提供单独的dialect类;例如，:func:`_sa.create_engine`URL“postgresql+psycopg2://@localhost/test”引用:mod:`psycopg2<.postgresql.psycopg2>`DBAPI / dialect组合，而URL“mysql+mysqldb://@localhost/test”引用:mod:`MySQL for Python<.mysql.mysqldb>`DBAPI / dialect组合。

    .. seealso::

        `PEP 249- Python数据库API规范v2.0<https://www.python.org/dev/peps/pep-0249/> `_ 

领域模型
    在问题解决和软件工程中，领域模型是与特定问题相关的所有主题的概念模型。它描述了各种实体，它们的属性，角色和关系，以及管控问题领域的任何约束条件。

    （来源：Wikipedia）

    .. seealso::

        `领域模型（通过维基百科）<https://en.wikipedia.org/wiki/Domain_model>`_ 

工作单元
    一种软件架构，在其中一个持久性系统（例如对象关系映射器）维护对一系列对象所做更改的列表，并在定期刷新所有这些待处理更改时将其全部提交到数据库中。

    SQLAlchemy的:class:`_orm.Session`实现了工作单元模式，通过使用:meth:`_orm.Session.add`等方法将对象添加到:meth:`_orm.Session`中，将参加到工作单位风格的持久性中。

    有关在SQLAlchemy中查看单元工作持久性是什么样子的演练，请从以下部分开始：:ref:`tutorial_orm_data_manipulation`。然后获取更多详细信息，请参见常规参考文档中的:ref:`session_basics`。

    .. seealso::

        “工作单元（通过Martin Fowler）<https://martinfowler.com/eaaCatalog/unitOfWork.html>`_ 

        :ref:`tutorial_orm_data_manipulation`

        :ref:`session_basics`

失效
已失效
失效时间
失效中
已失效的
    在SQLAlchemy ORM中，是指删除：term： `持久性`或有时：term： `分离的`对象中的数据，以便在下一次访问该对象的属性时，将发出:term：`懒惰加载` SQL查询以刷新此对象在当前进行的事务中存储的数据。

    .. seealso::

        :ref:`session_expire`

会话
    ORM数据库操作的容器或范围。会话从数据库中加载实例，跟踪映射实例的更改并在刷新时将更改持久化为单个工作单元。

    .. seealso::

        :doc:`orm/session`

列子句
    枚举要在结果集中返回的SQL表达式的“SELECT”语句的一部分。表达式直接跟随“SELECT”关键字，并是一个逗号分隔的单个表达式列表。

    例如：

    .. sourcecode:: sql

        SELECT user_account.name, user_account.email
        FROM user_account WHERE user_account.name = 'fred'

    上面的列列表"user_acount.name"，"user_account.email"是“SELECT”语句的列子句。

WHERE子句
    “SELECT”语句的一部分，用于指示应过滤哪些行的条件。它是跟随“WHERE”关键字的单个SQL表达式。

    .. sourcecode:: sql

        SELECT user_account.name, user_account.email
        FROM user_account
        WHERE user_account.name = 'fred' AND user_account.status = 'E'

    上面的短语“WHERE user_account.name ='fred' AND user_account.status ='E'”包括“SELECT”的WHERE子句。

FROM子句
    “SELECT”语句的一部分，用于指示行的初始源。

    简单的“SELECT”将在FROM子句中具有一个或多个表名。多个源由逗号分隔：

    .. sourcecode:: sql

        SELECT user.name, address.email_address
        FROM user, address
        WHERE user.id=address.user_id

    FROM子句还指定显式连接的位置。我们可以使用一条语句重写上面的“SELECT”，其中包含两个表的“JOIN”：

    .. sourcecode:: sql

        SELECT user.name, address.email_address
        FROM user JOIN address ON user.id=address.user_id

子查询
标量子查询
    引用嵌入在封闭“SELECT”中的“SELECT”语句。

    子查询分为两种一般类型之一，一种称为“标量选择”，该标量选择必须返回确切的一行一列，另一种类型被称为“派生表”，并用作来自另一个选择的FROM子句的行的来源。标量选择符合通过封闭语句给出的任何一行来选择的合适的情况，例如：WHERE子句，columns子句ORDER BY子句或HAVING子句，而派生表形式适用于封闭“SELECT”的FROM子句。

    示例：

    1.一个标量子查询放置在封闭“SELECT”的：term：`columns clause`中。该示例中的子查询是：term：`相关子查询`，因为给定的部分从中选择的行是通过封闭语句给出的：

    .. sourcecode:: sql

        SELECT id, (SELECT name FROM address WHERE address.user_id=user.id)
        FROM user

    2.标量子查询放置在封闭“SELECT”的WHERE子句中。此示例中的子查询未纠正，因为它选择一个固定结果。

    .. sourcecode:: sql

        SELECT id, name FROM user
        WHERE status=(SELECT status_id FROM status_code WHERE code='C')

    3.一个派生表子查询放置在封闭“SELECT”的FROM子句中。这样的子查询几乎总是获得别名。

    .. sourcecode:: sql

        SELECT user.id, user.name, ad_subq.email_address
        FROM
            user JOIN
            (select user_id, email_address FROM address WHERE address_type='Q') AS ad_subq
            ON user.id = ad_subq.user_id

关联
相关子查询
相关子查询
    如果:term：存在“SELECT”依赖于封闭“SELECT”的数据，则子查询为相关。在下面的示例中，子查询选择了来自“email_address”表的聚合值“MIN（a.id）”，以便针对“email_address.user_account_id”列与“user_account.id”列相关联的每个值触发子查询：

        SELECT user_account.name, email_address.email
         FROM user_account
         JOIN email_address ON user_account.id=email_address.user_account_id
         WHERE email_address.id = (
            SELECT MIN(a.id) FROM email_address AS a
            WHERE a.user_account_id=user_account.id
         )

    上述子查询引用“user_account”表，该表本身不在此嵌套查询的FROM子句中。相反，“user_account”表从封闭查询接收，其中从“user_account”选择的每个行都会导致子查询的不同执行。

    大多数情况下，相关子查询出现在直接封闭“SELECT”语句的WHERE子句或columns子句中，以及ORDER BY或HAVING子句中。

    在不常见的情况下，相关子查询可能存在于封闭SELECT的FROM子句中。在这些情况下，相关性通常是由封闭SELECT本身被封闭在WHERE子句，ORDER BY中的-columns或HAVING子句中而导致的。

    .. seealso::

        :term:`association relationship`

        :term:`relationship`

        :term:`one to many`

        :term:`many to one`

ACID
ACID模型
    ACID模型（“原子性，一致性，独立性，耐久性”的首字母）是保证数据库事务可靠处理的一组属性。
    (来源：Wikipedia)

    .. seealso::

        :term:`原子性`

        :term:`一致性`

        :term:`独立性`

        :term:`耐久性`

        `ACID模型（通过维基百科）<https://en.wikipedia.org/wiki/ACID_Model>`_ 

原子性
    原子性是:term：`ACID`模型的一个组成部分，并要求每个事务是“全有或全无”的：如果事务的一部分失败，则整个事务失败，数据库状态保持不变。原子系统必须保证在任何情况下包括断电，错误和崩溃在内的原子性。

    .. seealso::

        :term:`ACID`

        `原子性（通过维基百科）<https://en.wikipedia.org/wiki/Atomicity_(database_systems)>`_

一致性
    一致性是:term：`ACID`模型的一个组成部分，确保任何事务都会使数据库从一个有效状态转移到另一个有效状态。写入数据库的任何数据必须根据所有定义的规则是有效的，包括但不限于:term：`约束`，级联，触发器和所有组合。

    .. seealso::

        :term:`ACID`

        `一致性（通过维基百科）<https://en.wikipedia.org/wiki/Consistency_(database_systems)>`_

独立性
隔离性
已隔离
    :term:`ACID`模型的独立属性确保并发事务的并发执行结果会产生一个系统状态，该系统状态将与串行执行结果相同，即一个接一个的执行。每个事务必须在完全隔离的情况下执行，即，如果T1和T2并发执行，则T1和T2应保持独立。

    .. seealso::

        :term:`ACID`

        :term:`读未提交`

        :term:`读提交`

        :term:`可重复读取`

        :term:`可串行化`

可重复读
    四个数据库:term：`隔离`级别之一，可重复读具有:term：`读提交`的所有隔离性，并且另外具有读取事务中的特定行后，该行保证不会在该事务中由其他并发UPDATE语句发生更改（例如）。在该事务中，读取此行的对应结果保证是固定的。

读提交
    四个数据库:term：`隔离`级别之一，读提交特性是在事务中，事务不会暴露给未提交的并发事务的任何数据，防止所谓的“脏读取”。然而，在读提交下，可能会出现不可重复的读取，这意味着每一行的数据可能会更改，并且可能会在读取是，防止这种类型的情况是SQL标准中的标准的一个部分的:term：`可重复读`隔离级别。

读未提交
    四个数据库:term：`隔离`级别之一，读未提交特性是指在事务未提交时，对数据库数据进行的更改是在其他并发事务中可见的。但是，未提交的更改可能会以不一致的方式进行读取并依赖于根据时间戳或可序列化的事务隔离生成的结果。

可串行化
    四个数据库:term：`隔离`级别之一，可串行化具有:term：`可重复读取`的所有隔离性，并且保证在锁定基础上排除所谓的“幻像读取”；这意味着在该范围内插入或删除的行将在此事务中不可检测到。读取的行将保证继续存在，并且不存在不存在的行可以插入自另一个事务。

    可串行化隔离通常依靠锁定行或行范围来实现此效果，并且可能增加死锁的机会并降低性能。但是，还有非锁定式系统，但这些系统必然依赖于拒绝事务的写入冲突。

回归
过期
过期时间
失效中
已过期
    在SQLAlchemy ORM中，是指删除：term：`持久性`或有时：term：`分离的`对象中的数据，以便在下一次访问该对象的属性时，将发出:term：`懒惰加载` SQL查询以刷新此对象在当前进行的事务中存储的数据。

    .. seealso::

        :ref:`session_expire`

RETURNING
    这是某些后端提供的非SQL标准子句，以各种形式提供，用于在执行INSERT、UPDATE或DELETE语句时返回结果集。任何来自匹配行的列都可以返回，如同它们是从SELECT语句产生的一样。

    RETURNING子句为常见的更新/选择方案提供了巨大的性能提升，包括检索内联或默认生成的主键值和默认值在创建它们的时刻，以及以原子方式获取服务器生成的默认值。

    例如，面向PostgreSQL典型的RETURNING看起来像：

        INSERT INTO user_account (name) VALUES ('new name') RETURNING id, timestamp

    上述INSERT语句在执行时将提供包含列"user_account.id"和"user_account.timestamp"的结果集，如果没有包含，上述值本应生成为默认值（请注意，任何列的系列或SQL表达式都可以放在RETURNING中，而不仅仅是默认值列）。

    支持RETURNING或类似结构的后端是PostgreSQL、SQL Server、Oracle和Firebird。PostgreSQL和Firebird实现通常是全功能的，而SQL Server和Oracle的实现则有警告。在SQL Server上，该子句被称为“OUTPUT INSERTED”（对于INSERT和UPDATE语句）和“OUTPUT DELETED”（对于DELETE语句）；其中一个重要的警告是，触发器不支持与该关键字共同使用。在Oracle上，它被称为“RETURNING ...... INTO”，需要将值放入OUT参数中，这意味着不仅语法笨拙，而且一次只能用于一个行。

    SQLAlchemy的:meth:`。UpdateBase.returning`系统提供了在这些后端中处理返回列的一层抽象。ORM还包括许多优化，可以利用RETURNING。

一对多
    一种:func:`~sqlalchemy.orm.relationship`风格，将父映射器表的主键链接到相关表中的外键。每个唯一的父对象可以引用零个或多个唯一的相关对象。

    相关对象反过来将具有对其父对象的隐式或显式:term：`多对一`关系

    一个例子一对多模式（请注意，它与:term：`多对一`模式相同）：

    .. sourcecode:: sql

        CREATE TABLE department (
            id INTEGER PRIMARY KEY,
            name VARCHAR(30)
        )

        CREATE TABLE employee (
            id INTEGER PRIMARY KEY,
            name VARCHAR(30),
            dep_id INTEGER REFERENCES department(id)
        )

    从“department”到“employee”的关系是一对多，因为可以将许多员工记录与单个部门关联。一个SQLAlchemy映射可能看起来像：

        class Department(Base):
            __tablename__ = "department"
            id = Column(Integer, primary_key=True)
            name = Column(String(30))
            employees = relationship("Employee")


        class Employee(Base):
            __tablename__ = "employee"
            id = Column(Integer, primary_key=True)
            name = Column(String(30))
            dep_id = Column(Integer, ForeignKey("department.id"))

    .. seealso::

        :term:`relationship`

        :term:`many to one`

        :term:`backref`

多对一
    一种:func:`~sqlalchemy.orm.relationship`风格，它将父映射器表的外键链接到相关表的主键。每个父对象可以引用零个或一个相关对象。

    相关对象反过来将具有对任意数量的引用的隐式或显式:term：`一对多`关系，同时会引用它们的父对象。

    一个多对一的示例模式（请注意，它与:term：`一对多`模式相同）：

    .. sourcecode:: sql

        CREATE TABLE department (
            id INTEGER PRIMARY KEY,
            name VARCHAR(30)
        )

        CREATE TABLE employee (
            id INTEGER PRIMARY KEY,
            name VARCHAR(30),
            dep_id INTEGER REFERENCES department(id)
        )

    从“employee”到“department”的关系是多对一，因为许多员工记录可以与单个部门相关联。一个SQLAlchemy映射可能像这样：

        class Department(Base):
            __tablename__ = "department"
            id = Column(Integer, primary_key=True)
            name = Column(String(30))


        class Employee(Base):
            __tablename__ = "employee"
            id = Column(Integer, primary_key=True)
            name = Column(String(30))
            dep_id = Column(Integer, ForeignKey("department.id"))
            department = relationship("Department")

    .. seealso::

        :term:`relationship`

        :term:`one to many`

        :term:`backref`

反向引用
双向关系
    :term:`relationship`系统的扩展，其中两个不同的:func:`~sqlalchemy.orm.relationship`对象可以相互关联，以便它们随着任一侧的更改在内存中协调。这两个关系的最常见方法是使用:func:`~sqlalchemy.orm.relationship`函数显式为一侧指定，并指定“backref”关键字，以便另一个:func:`~sqlalchemy.orm.relationship`将自动创建。我们可以根据我们在:term:`一对多`中使用的示例说明如下：

        class Department(Base):
            __tablename__ = "department"
            id = Column(Integer, primary_key=True)
            name = Column(String(30))
            employees = relationship("Employee", backref="department")


        class Employee(Base):
            __tablename__ = "employee"
            id = Column(Integer, primary_key=True)
            name = Column(String(30))
            dep_id = Column(Integer, ForeignKey("department.id"))

    可以应用回引用到任何关系中，包括一对多，多对一和:term:`many to many` 。 

    .. seealso::

        :term:`relationship`

        :term:`one to many`

        :term:`many to one`

        :term:`many to many`

多对多
    将两个表通过中间的联合表链接在一起的:func:`~sqlalchemy.orm.relationship`风格。使用此配置，左侧的任意数量的行可以引用右侧的任意数量的行，反之亦然。

    以下图表所示，员工可以与项目相关联：

    .. sourcecode:: sql

        CREATE TABLE employee (
            id INTEGER PRIMARY KEY,
            name VARCHAR(30)
        )

        CREATE TABLE project (
            id INTEGER PRIMARY KEY,
            name VARCHAR(30)
        )

        CREATE TABLE employee_project (
            employee_id INTEGER PRIMARY KEY,
            project_id INTEGER PRIMARY KEY,
            FOREIGN KEY employee_id REFERENCES employee(id),
            FOREIGN KEY project_id REFERENCES project(id)
        )

    上面的“employee_project”表是多对多表，自然形成由来自每个相关表的主键组成的组合主键。

    在SQLAlchemy中，:func:`sqlalchemy.orm.relationship`函数可以用不带任何缀的普通表元数据表示该风格的关系：

        class Employee(Base):
            __tablename__ = "employee"

            id = Column(Integer, primary_key=True)
            name = Column(String(30))

            projects = relationship(
                "Project",
                secondary=Table(
                    "employee_project",
                    Base.metadata,
                    Column("employee_id", Integer, ForeignKey("employee.id"), primary_key=True),
                    Column("project_id", Integer, ForeignKey("project.id"), primary_key=True),
                ),
                backref="employees",
            )


        class Project(Base):
            __tablename__ = "project"

            id = Column(Integer, primary_key=True)
            name = Column(String(30))

    上面的“Employee.projects”和反向引用的“Project.employees”集合被定义：

        proj = Project(name="Client A")

        emp1 = Employee(name="emp1")
        emp2 = Employee(name="emp2")

        proj.employees.extend([emp1, emp2])

    .. seealso::

        :term:`关联关系`

        :term:`relationship`

        :term:`one to many`

        :term:`many to one`

    )。

关系
多重关系
    两个映射类之间的连接单元，对应于数据库中这两个表之间的某些关系。

    使用SQLAlchemy函数:func:`~sqlalchemy.orm.relationship`定义关系。一旦创建，SQLAlchemy会检查所涉及的映射和基础映射表以分类关系为三种类型之一：:term:`一对多`，:term:`多对一`或:term:`多对多`。通过这种分类，关系构造处理将在响应内存中的对象关联的更改的情况下，将适当的链接持久性导入到数据库中，以及基于当前链接在内存中将对象引用和集合加载到内存中。关系与:term:`association relationship`不同，后者使用中间关联表将两个表连接在一起。

    .. seealso::

        :ref:`relationship_config_toplevel`

游标
    一种控制结构，可在数据库中遍历记录。在Python DBAPI中，游标对象实际上是语句执行的起点，以及用于获取结果的接口。

    .. seealso::

        `游标对象（在pep-249中）<https://www.python.org/dev/peps/pep-0249/#cursor-objects>`_

        `游标（通过维基百科）<https://en.wikipedia.org/wiki/Cursor_(databases)>`_employee_id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    project_id = Column(Integer, ForeignKey("project.id"), primary_key=True)
    role_name = Column(String(30))

    project = relationship("Project", backref="project_employees")
    employee = relationship("Employee", backref="employee_projects")

    员工可以根据角色名称添加到项目中：

        proj = Project(name="Client A")

        emp1 = Employee(name="emp1")
        emp2 = Employee(name="emp2")

        proj.project_employees.extend(
            [
                EmployeeProject(employee=emp1, role_name="技术主管"),
                EmployeeProject(employee=emp2, role_name="账户主管"),
            ]
        )

    .. seealso::

        :term:`多对多`

    constraint
    constraints
    constrained
        关系数据库中的规则，确保数据的有效性和一致性。常见约束包括：:term:`主键约束`、:term:`外键约束`和:term:`检查约束`。

    candidate key

        :term:`关系代数`术语，指属性或属性集，可用于行的唯一标识键/索引。一行可能有多个候选键，每个键都适用于该行的主键。表的主键总是一个候选键。

        .. seealso::

            :term:`主键`

            `候选键（via 维基百科）<https://en.wikipedia.org/wiki/Candidate_key>`_

            https://www.databasestar.com/database-keys/

    primary key
    primary key constraint

        :term:`约束`，唯一定义表中每一行的特征。主键必须由任何其他行无法重复的特征组成。主键可以由单个属性或多个属性组合成。创建表时通常（但不总是）定义主键，如下所示：

        .. sourcecode:: sql

            CREATE TABLE employee (
                 emp_id INTEGER,
                 emp_name VARCHAR(30),
                 dep_id INTEGER,
                 PRIMARY KEY (emp_id)
            )

        .. seealso::

            :term:`复合主键`

            `主键（via 维基百科）<https://en.wikipedia.org/wiki/Primary_Key>`_

    composite primary key

        有多个列的 :term:`主键`。根据两个或更多列而不仅仅是单个值，可以唯一识别特定的数据库行。

        .. seealso::

            :term:`主键`

    foreign key constraint
        两个表之间的参照约束。外键是关系型表中一个或多个字段，与另一个表的一组 :term:`候选键`匹配。外键可用于交叉引用表。

        .. via 维基百科定义:

        可以使用以下SQL标准 DDL 将外键约束添加到表中：

        .. sourcecode:: sql

            ALTER TABLE employee ADD CONSTRAINT dep_id_fk
            FOREIGN KEY (employee) REFERENCES department (dep_id)

        .. seealso::

            `外键约束 (via 维基百科) <https://en.wikipedia.org/wiki/Foreign_key_constraint>`_

    check constraint

        检查约束是指定添加或更新关系型数据库表中条目时定义有效数据的条件。检查约束适用于表中的每一行。

        .. via 维基百科定义:

        可以使用以下SQL标准 DDL 添加检查约束到表中：

        .. sourcecode:: sql

            ALTER TABLE distributors ADD CONSTRAINT zipchk CHECK (char_length(zipcode) = 5);

        .. seealso::

            `检查约束 (via 维基百科) <https://en.wikipedia.org/wiki/Check_constraint>`_

    unique constraint
    unique key index
        唯一键索引可以唯一识别数据库表中每行数据值。唯一键索引包括单个列或单个数据库表中一组列。如果不使用 NULL 值，则不同的行或数据记录在这些单一键索引列中不能具有相同的数据值或数据值组合。根据其设计，数据库表可以有许多唯一键索引，但最多只能有一个主键索引。

        .. via 维基百科定义:

        .. seealso::

            `唯一约束 (via 维基百科) <https://en.wikipedia.org/wiki/Unique_key#Defining_unique_keys>`_

    transient
        这描述了一个 :term:`Session` 中一个对象可以具有的主要对象状态之一。暂态对象是一个新对象，没有任何数据库身份识别，并且尚未与会话关联。将对象添加到会话后，它将转移到 :term:`pending` 状态。

        .. seealso::

            :ref:`session_object_states`

    pending
        这描述了一个 :term:`Session` 中一个对象可以具有的主要对象状态之一。挂起对象是一个新对象，没有任何数据身份识别，但最近已与会话关联。当会话发出刷新并插入行时，对象将移至 :term:`persistent` 状态。

        .. seealso::

            :ref:`session_object_states`

    deleted
        这描述了一个 :term:`Session` 中一个对象可以具有的主要对象状态之一。已删除的对象是先前持久性的对象，已经在刷新中向数据库发出了 DELETE 表示其行已被删除。一旦会话的事务提交，该对象将移至 :term:`detached`状态；或者，如果回滚了会话的事务，则会撤消 DELETE 并将对象移回到 :term:`persistent` 状态。

        .. seealso::

            :ref:`session_object_states`

    persistent
        这描述了一个 :term:`Session` 中一个对象可以具有的主要对象状态之一。持久对象是具有数据库身份（即主键）并当前与会话关联的对象。任何处于 :term:`pending` 状态并已被插入的对象都处于持久状态，任何在会话中从数据库中加载的对象都处于持久状态。将持久对象从会话中删除后，它将变为 :term:`detached`。

        .. seealso::

            :ref:`session_object_states`

    detached
        这描述了一个 :term:`Session` 中一个对象可以具有的主要对象状态之一。已分离的对象是具有数据库身份（即主键）但未与任何会话关联的对象。先前是 :term:`persistent` 并已从其会话中移除的对象通过清除或关闭所属会话而进入分离状态。当对象在会话之间移动时，或从外部对象缓存移动时，通常使用分离状态。

        .. seealso::

            :ref:`session_object_states`

    attached
        表示当前与特定 :term:`Session` 关联的 ORM 对象。

        .. seealso::

            :ref:`session_object_states`