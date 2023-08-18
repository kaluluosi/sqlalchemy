.. _glossary:

========
术语表
========

.. glossary::
    :sorted:

    1.x 风格
    2.0 风格
    1.x-风格
    2.0-风格
        这些术语是SQLAlchemy 1.4中新增的, 与SQLAlchemy 1.4中到2.0之间的过渡计划有关, 描述详见   :ref:`migration_20_toplevel` 。
        "1.x 风格"指使用一种API的方式, 它在1.x系列及以前的版本中已被文档化, 如1.3, 1.2等, 而"2.0 风格"指API在2.0版本中的呈现方式。
        SQLAlchemy Version 1.4在所谓的“过渡模式”中（即实现了几乎2.0的所有API），而版本2.0仍保留了   :class:`_orm.Query`  对象以允许遗留代码基本保持2.0兼容性。

        .. seealso::

              :ref:`migration_20_toplevel` 

    sentinel
    插入哨兵
        这是SQLAlchemy特有的术语，用于跟踪通过RETURNING或类似方法传回的行以便用于大量INSERTmanyvalues操作的  :term:`_schema.Column`  列之一.
        此类列配置对于INSERTmanyvalues功能通常自动使用代理整数主键列作为“插入哨兵”，不需要用户进行配置。
        少数情况下有其他类型的服务器生成的主键值，在  :term:`table metadata`  中可以选择显式配置"insert sentinel"列以优化同时插入多行的INSERT语句。

        .. seealso::

              :ref:`engine_insertmanyvalues_returning_order`  - 在   :ref:` engine_insertmanyvalues`  节中

    多个值的插入
        这是SQLAlchemy特有的功能，允许INSERT语句在单个语句中发出数千个新行，同时允许使用RETURNING或类似语句内联地返回服务器生成的值，以优化性能。
        该功能旨在为选择的后端透明地提供支持，但确实提供了一些配置选项。有关此功能的完整说明，请参见   :ref:`engine_insertmanyvalues`  节。

        .. seealso::

              :ref:`engine_insertmanyvalues` 

    mixin类
    mixin类
        一种常见的面向对象模式，其中一个类包含可供其他类使用的方法或属性，而无需成为那些其他类的父类。

        .. seealso::

            `Mixin ( 维基百科) <https://zh.wikipedia.org/wiki/%E6%8F%92%E5%85%A5_(%E9%9D%A2%E5%90%91%E5%AF%B9%E8%B1%A1%E7%BC%96%E7%A8%8B)>`_


    反射
    反射的
        在SQLAlchemy中，此术语指查询数据库架构目录以加载有关现有表格、列、限制和其他构造的信息的功能。SQLAlchemy包括可以提供这些信息的原始数据的功能，以及可以自动从数据库架构目录中构建Core / ORM可用的   :class:`.Table`  对象的功能。

        .. seealso::

              :ref:`metadata_reflection_toplevel`  - 关于数据库反射的完整背景

              :ref:`orm_declarative_reflected`  - 关于集成ORM映射和反射表的背景


    命令式
    声明式
        在SQLAlchemy ORM中，这些术语指将Python类映射到数据库表的两种不同的风格。

        .. seealso::

              :ref:`orm_imperative_mapping` 

              :ref:`orm_declarative_mapping` 

    门面
        作为一个前置界面的对象，掩盖了更复杂的底层或结构性代码。

        .. seealso::

            `门面模式 ( 维基百科) <https://zh.wikipedia.org/wiki/%E9%97%A8%E9%9D%A2%E6%A8%A1%E5%BC%8F>`_

    关系
    关系代数
        由Edgar F. Codd开发的代数系统，用于建模和查询存储在关系数据库中的数据。

        .. seealso::

            `关系代数 ( 维基百科) <https://zh.wikipedia.org/wiki/%E5%85%B3%E7%B3%BB%E4%BB%A3%E6%95%B0>`_

    笛卡尔积
        给定两个A和B的集合, 笛卡尔积是所有有序对(a, b)的集合, 其中a在A中, b在B中。

        就SQL数据库而言, 当我们选择两个或多个表(或其他子查询)而没有建立任何形式的条件
        （直接或间接）来联系一个表的行与另一个表的行时，就会出现笛卡尔积。 如果我们同时从表 A 和表 B 接收一个 SELECT，我们会得到每一个 A 的行与 B 的第一行相匹配，然后每个 A 的行都与 B 的第二行相匹配，以此类推，直到 A 的每行都与 B 的每行配对为止。

        笛卡尔积会生成巨大的结果集，并且如果不进行防止措施容易导致客户端应用程序崩溃。

        .. seealso::

            `笛卡尔积 ( 维基百科) <https://zh.wikipedia.org/wiki/%E7%AC%9B%E5%8D%A1%E5%B0%94%E7%A7%AF>`_

    圈度复杂度
        表示源代码中可能路径的数量的代码复杂度度量。

        .. seealso::

            `圈度复杂度 ( 维基百科) <https://zh.wikipedia.org/wiki/%E5%9C%88%E5%A4%8D%E5%BC%8F%E5%A4%8D%E6%9D%82%E5%BA%A6>`_

    绑定参数
    绑定参数们
    绑定的参数
    绑定参数化
        绑定参数是将数据传递到  :term:`DBAPI`  数据库驱动程序的主要手段。
        虽然要调用的操作基于SQL语句字符串，但是数据值本身
        是分别传递的，其中驱动程序包含了将安全地处理这些字符串并将它们传递
        到后端数据库服务器的逻辑，这可能要么涉及将参数格式化到SQL字符串本身中，
        要么使用单独的协议传递它们到数据库。

        数据库驱动程序执行此操作的特定系统应不重要对于调用方来说;关键点
        是，在外部，数据应始终分别传递而不是作为SQL字符串的一部分传递。
        这对于有充足防止SQL注入漏洞的安全性以及帮助驱动程序具备最佳性能都至关重要。

        .. seealso::

            `准备执行语句 (via 维基百科) <https://en.wikipedia.org/wiki/Prepared_statement>`_ - 维基百科

            `bind parameters (via Index鸟哥) <https://use-the-index-luke.com/sql/where-clause/bind-parameters>`_ - Index鸟哥

              :ref:`tutorial_sending_parameters`  - 在   :ref:` unified_tutorial`  中


    可选择
        在SQLAlchemy中， "可选择"（selectable）是用于表示一组行的SQL构造的术语。它与“关系”在  :term:`关系代数`  的概念上相似。
        在SQLAlchemy中，当使用SQLAlchemy Core使用时，子类化   :class:`_expression.Selectable`  类的对象被认为是可作为"user-defined"表的 "selectables"。
        最常见的两个结构是   :class:`_schema.Table`  和   :class:` _expression.Select`  语句。

    ORM 注释
    注释们
        短语“ORM注释”指的是SQLAlchemy的一个内部方面，其中一个Core对象(如  :class:`.Table` ，  :class:` .Column` .Select`对象)
        可以携带附加的运行时信息，标记它属于特定ORM映射。该术语不应与需要
        静态类型信息的Python代码源“类型注释”这一常见短语混淆. 大多数SQLAlchemy的
        文档代码示例都是带有“带注释的示例”或“未注释的示例”的小注释。

        当文档中出现"ORM注释"的短语时，它指的是CoreSQL表达式对象，如  :class:`.Table` ，  :class:` .Column` .Select`对象，
        这些都源于一个或多个ORM映射，并因此在传递给ORM对象的方法（例如  :meth:`_orm.Session.execute`  ）
        时会有ORM特定的解释和/或行为。例如，当我们从ORM映射构造 :class:`.Select` 对象时，
        如ORM教程<ref>`ORM教程 <tutorial_declaring_mapped_classes>`_所示的“User”类

        例如：

            ">>> stmt = select(User)

        上述  :class:`.Select` .Table` ，
        但没有直接引用“User”类。这是   :class:`.Select`  构建仍保持与Core级别的过程兼容的方式
        （注意，   :class:`.Select` ` ._raw_columns`` 成员是私有的，不应由最终用户代码访问）::

            >>> stmt._raw_columns
            [Table('user_account'MetaData(), Column('id', Integer(), ...)]

        但是，当我们的   :class:`.Select`  传递给ORM的   :class:` .Session`  后，间接与该对象相关的ORM实体用于
        在ORM上下文中解释   :class:`.Select` ，支持多表选择。实际的"ORM注释"可以在另一个私人变量:` `._annotations`` 中看到::

            >>> stmt._raw_columns[0]._annotations
            immutabledict({
              'entity_namespace': <Mapper at 0x7f4dd8098c10; User>,
              'parententity': <Mapper at 0x7f4dd8098c10; User>,
              'parentmapper': <Mapper at 0x7f4dd8098c10; User>
              })

        因此，我们称 ``stmt`` 为 **ORM注释的 select()** 对象。它是一个   :class:`.Select`  语句，包含附加信息，
        以在ORM特定的方式中解释在像ORM映射传递给之后的代码  :meth:`_orm.Session.execute`   方法中。

    插件
    插件已启用
    插件特定
        "插件已启用"或"插件特定"通常指SQLAlchemy Core中的一个函数或方法，在ORM上下文中使用时将有所不同。

        SQLAlchemy允许Core构造，例如 :class:`_sql.Select` 对象，参与“插件”系统，该插件可以注入其他
        没有默认存在的行为和功能。

        具体来说，主要的"插件"是"ORM插件"，它处于构成SQLAlchemy ORM所使用的系统的基础。
        可以使用Core检索到的方法来构建和执行SQL查询，以返回ORM结果。

        .. seealso::

              :ref:`migration_20_unify_select` 

    增删改查
    CRUD
        一个缩写，表示"创建（Create）、更新（Update）、删除（Delete）"。 在SQL中，该术语指的是执行三个广泛认识的操作：`INSERT`、`UPDATE`和`DELETE`语句。

    多值执行
        此术语指  :pep:`249`  DBAPI 规范的一部分，该规范指出可以在多个参数集的情况下对数据库连接执行单个 SQL 语句。
        具体的方法称为 `cursor.executemany() <https://peps.python.org/pep-0249/#executemany>`_，并且它与用于单个语句调用的
        `cursor.execute() <https://peps.python.org/pep-0249/#execute>`_ 方法具有许多行为差异。 "executemany" 方法会多次执行
        给定的SQL语句，每次传递一个参数集。使用executemany的一般原因是改善性能，在其中数据库API可使用预处理的SQL一次性执行
        会有很好的表现，或者为多次调用相同语句进行优化。

        当传递包含多个参数字典的参数列表时，SQLAlchemy通常会自动使用 "cursor.executemany()" 方法。这表明SQL语句及其处理后的参数集
        应传递给 "cursor.executemany()"，在该方法中，语句将对每个参数字典单独依次执行。

        使用所有已知DBAPI的cursor.executemany()方法的一个关键限制是，当使用executemany()方法时，cursor未配置为返回行。 对于
        **大多数** 后端数据库 (在此情况下的一个重要例外是cx_Oracle / OracleDB DBAPIs)，这意味着“INSERT..RETURNING”之类的语句通常不能使用
        直接使用 `cursor.executemany()`，因为DBAPI通常不会将每个INSERT执行的单个行聚合在一起。

        为了克服此限制，从2.0系列起，SQLAlchemy实现了另一种称为   :ref:`engine_insertmanyvalues`  的"executemany"形式。
        此功能使用 ``cursor.execute()`` 调用插入语句，该语句将在一次交互中处理多个参数集，因此产生与使用 ``cursor.executemany()`` 相同的效果，同时支持 RETURNING。

        .. seealso::

              :ref:`tutorial_multiple_parameters`  - 关于“executemany”的教程介绍

              :ref:`engine_insertmanyvalues`  - SQLAlchemy功能，允许使用"executemany"使用RETURNING

    数据马歇尔
    数据对象映射
        转换对象的内存表示形式以适合于存储或传输到系统的另一部分时所使用的过程, 需要移动计算机程序的不同部分或从一个程序到另一个程序时使用。
        在SQLAlchemy中，我们通常需要将数据转换为适合于传递到关系数据库的格式, 称为"数据对象映射"(Data Object Mapping)。

        .. seealso::

            `数据马歇尔（维基百科） <https://zh.wikipedia.org/wiki/序列化>`_

              :ref:`types_typedecorator`  - SQLAlchemy 的   :class:` .TypeDecorator`  通常用于数据对象映射，当数据发送到数据库的 INSERT 和 UPDATE 语句时会用到，并在 SELECT 语句中检索（unmarshalling）数据时进行"unmarshalling"

    描述符
    描述符
        在 Python 中，描述符是具有“绑定行为”的对象属性，其属性访问已被方法覆盖 在描述符 协议 中。 这些方法是 __get__()，__set__() 和 __delete__()。
        如果为对象定义了其中任何方法，则称其为描述符。

        在SQLAlchemy ORM中，使用   :class:`_orm.Mapper`  类的实例将一类映射到数据库表时，描述符用于提供ORM属性行为。
        当与数据库列或相关联的对象引用的属性被访问时，ORM使用与该属性对应的描述符触发 ORM：  :class:`.Session`  和生命周期事件。

    标识键
        ORM映射对象的关键标识符。标识键与ORM映射对象的主键标识器及其在   :class:`_orm.Session`  的  :term:` identity map`  中的惟一标识相对应。

        在SQLAlchemy中，可以使用   :func:`_sa.inspect`  API检查ORM对象的标识键，以返回   :class:` _orm.InstanceState`  跟踪对象，然后查看  :attr:`_orm.InstanceState.key`  属性::

            >>> from sqlalchemy import inspect
            >>> inspect(some_object).key
            (<class '__main__.MyTable'>, (1,), None)

        .. seealso::

            :term:`identity map` 

    标识映射
        Python对象和其数据库标识之间的映射。标识映射是与ORM  :term:`session`  关联的集合，它将每个数据库对象的单个实例与其标识键关键字配对。
        此模式的优点是，针对特定数据库标识的所有操作都在单个对象实例上透明地完成。在ORM  :term:`session`  与  :term:` isolated`  事务结合使用时，拥有具有特定主键的对象的引用从实际上是代理实际数据库行的套接字的角度来看。

        .. seealso::

            `Identity Map (via Martin Fowler) <https://martinfowler.com/eaaCatalog/identityMap.html>`_

              :ref:`session_get`  - 通过主键在标识映射中查找对象的方法

    延迟初始化
        推迟某些初始化操作，例如创建对象，填充数据或与其他服务建立连接，直到需要这些资源。

        .. seealso::

            `延迟初始化( via 维基百科 )  <https://zh.wikipedia.org/wiki/延迟初始化>`_

    惰性加载
    惰性加载们
    惰性加载过的
    惰性加载
        在对象关系映射中，"惰性加载（lazy load）"是指一个属性在一段时间内不包含其数据库端值，通常是在对象最初加载时。而是在首次使用该属性时，会收到“缓存标记”，以便在第一次使用该属性时从数据库加载数据。使用此模式，可以有时减小对象提取中的复杂性和时间，因为与相关表的相关属性不需要立即处理。
        惰性加载是  :term:`急切加载`  的相反方法。

        在SQLAlchemy中，  :term:`惰性加载`  是ORM的一个核心功能，并应用于用户定义类上具有映射属性的属性。 当引用指向数据库列或相关对象的属性且未加载值时，
        ORM使用当前对象在  :term:`持久`  状态下与之关联的   :class:` _orm.Session` ，在当前事务中发出SELECT语句，如果没有进程当中有事务的话,
        将新开一个事务。如果对象处于  :term:`分离`  状态，并且未与任何   :class:` _orm.Session`  关联，这被认为是一种错误状态，并将触发有关该状态的确切异常。

        .. seealso::

            `惰性加载（via Martin Fowler） <https://martinfowler.com/eaaCatalog/lazyLoad.html>`_

             :term:`N个加一千问题` 

              :ref:`loading_columns`  - 包括 ORM 映射的列的惰性加载信息

             :doc:`orm/queryguide/relationships`  - 包括 ORM 相关对象的惰性加载信息

              :ref:`asyncio_orm_avoid_lazyloads`  - 有关在使用   :ref:` asyncio_toplevel`  扩展时避免惰性加载的提示

    急切加载
    急切加载们
    急切加载上的
    急切加载
        在对象关系映射中， "急切加载" 指在对象本身从数据库加载时同时将属性和集合填充到其数据库端值。在 SQLAlchemy 中，术语 "急切加载" 通常
        指使用   :func:`_orm.relationship`  构建关系时会将相关集合和对象的实例通过 ORM 映射相互链接，但还可以指从其他表格属性中加载带有其他列属性的，例如继承映射。

        急切加载与  :term:`惰性加载`  相反。

        .. seealso::

             :doc:`orm/queryguide/relationships` 

    映射
    已映射
    映射的类
    ORM 映射类
        我们在将类与   :class:`_orm.Mapper`  类的实例关联时说该类 "已映射"（mapped），这个过程为类与数据库表或其他  :term:` selectable`  构造相关联，以便可以使用   :class:`.Session`  对象来持久化和加载其实例。

        .. seealso::

              :ref:`orm_mapping_classes_toplevel` 

    N个加一千问题
    N个加一千
        N个加一千问题是  :term:`懒惰加载`  模式的常见副作用，其中应用程序希望迭代每个对象结果集上的相关属性或集合，在其中，该属性或集合被设置为通过惰性加载策略加载。
        结果是已经发出 SELECT 语句来加载所需的基本对象结果集，然后，随着应用程序遍历每个成员，为了为该成员加载相关属性或集合，为每个成员发出额外的 SELECT 语句.
        在N个成员的结果集中，会发出N+ 1个SELECT语句。

        N个加一千问题通过  :term:`急切加载`  得到解决。

        .. seealso::

              :ref:`tutorial_orm_loader_strategies` 

             :doc:`orm/queryguide/relationships` 

    多态的
    多态化地
        在SQLAlchemy中，此术语指处理同时处理几种类型的功能。通常情况下，这个术语应用于ORM映射的概念，通过查询操作返回不同的子类，根据结果集中的信息来检查使用  :term:`鉴别符`  的值，通常是一个列。

        SQLAlchemy中的多态化加载意味着使用三种不同方案之一或结合使用三种不同类型映射来映射层次结构; "联接"，"单表" 和 "具体"。 模块   :ref:`inheritance_toplevel`  具体描述了继承映射。

    方法链接生成的
    “方法链接”，在SQLAlchemy文档中也称为“生成”，是一种面向对象的技术，通过在对象上调用方法构造对象的状态。对象包含任意数量的方法，每个方法返回添加到对象中的新对象（或在某些情况下是相同的对象）。

    在SQLAlchemy中使用最多的两个对象是  :class:`_expression.Select` .orm.query.Query` 对象。例如，通过调用：meth:`_expression.Select.where` 和 :meth:`_expression.Select.order_by` 方法，可以将两个表达式分配给 “WHERE” 子句以及一个 “ORDER BY” 子句::

        stmt = (
            select(user.c.name)
            .where(user.c.id > 5)
            .where(user.c.name.like("e%"))
            .order_by(user.c.name)
        )

    上述每个方法调用都会返回原始的 :class:`_expression.Select` 对象的副本，并添加其他的限定词。

发布
发行版本
发布的
    在SQLAlchemy中，“发布”的术语是指结束对特定数据库连接的使用的过程。 SQLAlchemy支持连接池的使用，这允许配置数据库连接的寿命。使用连接池连接时，“关闭”它的过程，即调用类似于“连接.关闭()”的语句，可能会使该连接返回到现有池中，也可能会使其实际关闭由该连接引用的底层TCP / IP连接 - 这取决于当前池的配置以及池的当前状态。 因此，我们使用"released"这个术语，表示“我们在结束使用它们时做任何我们想做的连接”。有时该术语将用于短语“释放事务资源”，以更明确地指出我们实际上“释放”的是连接上积累的任何事务状态。在大多数情况下，从表中选择、发出更新等操作在该连接上获得  :term:`隔离状态`  以及可能的行或表锁。这种状态全部局限于连接上的特定事务，并在我们发出回滚时释放。连接池的一个重要功能是，当我们将连接返回到池中时，也会调用DBAPI的“connection.rollback()”方法，以便在准备再次使用连接时，它将处于不带有前面一系列操作的“清理”状态。：

        .. seealso::

              :ref:`pooling_toplevel` 

    DBAPI
    pep-249
        DBAPI是“Python数据库API规范”的缩写。这是Python中广泛使用的规范，用于定义所有数据库连接包的常用使用模式。 DBAPI是一个“低级” API，通常是应用程序中用于与数据库通信的最低级系统。SQLAlchemy的  :term:`dialect`  系统是围绕DBAPI的操作构建的，它提供服务于DBAPI的独立方言类，这些类位于特定的数据库引擎之上；例如，  :func:` _sa.create_engine`  DBAPI/方言组合，而URL“mysql+mysqldb://@localhost/test”则引用了  :mod:`MySQL for Python <.mysql.mysqldb>`  DBAPI/方言组合。

        .. seealso::

            `PEP 249-Python Database API Specification v2.0 <https://www.python.org/dev/peps/pep-0249/>`_

    领域模型
        在问题解决和软件工程中，领域模型是与特定问题相关的所有主题的概念模型。它描述了各种实体、它们的属性、角色和关系，以及管理问题域的约束条件。（来源：维基百科）

        .. seealso::

            “领域模型（通过维基百科）<https://en.wikipedia.org/wiki/Domain_model>`_

    工作单元
        一种软件架构，其中一个持久性系统（例如ORM）维护对一系列对象所做的更改列表，并在定期将所有挂起的更改刷新到数据库时执行。

        SQLAlchemy的  :class:`_orm.Session`  这样的方法添加到 :class:` _orm.Session`的对象将参与单位-of-work样式的持久化。

        关于单位工作持久化的演练，可以从  :ref:`unified_tutorial` 。

        .. seealso::

            “单位工作（通过Martin Fowler）<https://martinfowler.com/eaaCatalog/unitOfWork.html>`_

              :ref:`tutorial_orm_data_manipulation` 

              :ref:`session_basics` 

    过期的
    过期
    过期的
    即将过期的
        在SQLAlchemy ORM中，指的是将数据从  :term:`persistent`  对象或有时  :term:` detached`  对象中擦除的过程，因此，在下次访问对象的属性时，将发出  :term:`lazy load`   SQL查询，以刷新当前正在执行操作的事务中该对象存储的数据。

        .. seealso::

              :ref:`session_expire` 

    会话
        ORM数据库操作的容器或范围。 会话从数据库中加载实例，跟踪对映射实例的更改并在刷新时将更改持久化为单个工作单元。

        .. seealso::

             :doc:`orm/session` 

    columns子句
        枚举在结果集中要返回的SQL表达式的“SELECT”语句部分。这些表达式直接跟随“SELECT”关键字，并且是由标准逗号分隔的单个表达式列表。

        例如：

        .. sourcecode:: sql

            SELECT user_account.name, user_account.email
            FROM user_account WHERE user_account.name = 'fred'

        上述“user_acount.name”和“user_account.email”之类的字段/表达式是“SELECT”的columns子句。

    WHERE子句
        用于指示要过滤哪些行的“SELECT”语句部分。它是一个跟随关键字“WHERE”的单个SQL表达式。

        .. sourcecode:: sql

            SELECT user_account.name, user_account.email
            FROM user_account
            WHERE user_account.name = 'fred' AND user_account.status = 'E'

        上述短语“WHERE user_account.name = 'fred' AND user_account.status = 'E'”是“SELECT”的WHERE子句。

    FROM子句
        指示所选行的初始来源的“SELECT”语句部分。

        简单的“SELECT”将在FROM子句中包含一个或多个表名称。多个源由逗号分隔：

        .. sourcecode:: sql

            SELECT user.name, address.email_address
            FROM user, address
            WHERE user.id=address.user_id

        “FROM”子句也是指定显式连接的位置。我们可以使用一个包含“JOIN”表的单个“FROM”元素来重写上面的“SELECT”：

        .. sourcecode:: sql

            SELECT user.name, address.email_address
            FROM user JOIN address ON user.id=address.user_id


    子查询
    标量子查询
        指的是嵌套在一个封闭的“SELECT”中的“SELECT”语句。

        子查询有两种一般的形式，一种是称为“标量选择”的特定形式，该形式必须返回恰好一个行和一列，另一种是充当“派生表”的形式，作为另一个查询的“FROM”子句的行源。 标量选择申请
        可以在封闭的“SELECT”的WHERE子句、columns子句、ORDER BY子句或HAVING子句中，而派生表形式可以在封闭“SELECT”的FROM子句中放置。

        例子：

        1.将标量子查询放置在封闭“SELECT”的columns子句中。在此示例中，子查询是一:T作为“关联子查询”的一部分，因为它选择的行的一部分是由封闭语句提供的。

        .. sourcecode:: sql

            SELECT id, (SELECT name FROM address WHERE address.user_id=user.id)
            FROM user

        2. 将标量子查询放在封闭“SELECT”的WHERE子句中。在此示例中，此子查询不是关联的，因为它选择了固定结果。

        .. sourcecode:: sql

            SELECT id, name FROM user
            WHERE status=(SELECT status_id FROM status_code WHERE code='C')

        3. 将派生表子查询放置在封闭“SELECT”的FROM子句中。这种子查询几乎总是得到一个别名。

        .. sourcecode:: sql

            SELECT user.id, user.name, ad_subq.email_address -- ...etc...
            FROM
                user JOIN
                (select user_id, email_address FROM address WHERE address_type='Q') AS ad_subq
                ON user.id = ad_subq.user_id

    关联
    关联子查询
    相关子查询
        如果一个子查询依赖于封闭“SELECT”的数据，则称其为相关。下面是一个子查询，“SELECT”语句从中选择“email_address”表的聚合值“MIN(a.id)”，因此它将为“user_account.id”的每个值调用一次：

        .. sourcecode:: sql

            SELECT user_account.name, email_address.email
             FROM user_account
             JOIN email_address ON user_account.id=email_address.user_account_id
             WHERE email_address.id = (
                SELECT MIN(a.id) FROM email_address AS a
                WHERE a.user_account_id=user_account.id
             )

        上述子查询引用了“user_account”表，该表不在此嵌套查询的“FROM”子句中。相反，“user_account”表从封闭查询中接收，其中从“user_account”中选择的每行都会导致子查询的不同执行。

        一个关联的子查询在大多数情况下存在于封闭的“SELECT”的WHERE子句或columns子句中，以及ORDER BY或HAVING子句中。

        在较少的情况下，相关子查询可能存在于封闭的“SELECT”的FROM子句中。在这些情况下，相关通常是由封闭的“SELECT”本身包含在WHERE、ORDER BY、columns或HAVING子句中的另一个“SELECT”而来。

        .. sourcecode:: sql

            SELECT parent.id FROM parent
            WHERE EXISTS (
                SELECT * FROM (
                    SELECT child.id AS id, child.parent_id AS parent_id, child.pos AS pos
                    FROM child
                    WHERE child.parent_id = parent.id ORDER BY child.pos
                LIMIT 3)
            WHERE id = 7)

        直接从一个“SELECT”沿着FROM子句到另一个“SELECT”的关联不可能，因为关联只能在从封闭语句的FROM子句中的源行可用时进行。

    ACID
    ACID模型
        一组特性，可保证数据库事务的可靠处理。“原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）和耐久性（Durability）”（通过维基百科）

        .. seealso::

             :term:`atomicity` 

             :term:`consistency` 

             :term:`isolation` 

             :term:`durability` 

            “ACID模型（通过维基百科）<https://en.wikipedia.org/wiki/ACID_Model>`_

    原子性
        ACID模型的四个组成部分之一，要求每个事务都是“要么全部做完，要么全部不做”：如果事务的任何一部分失败，则整个事务失败，并且数据库状态保持不变。 原子系统必须保证在任何情况下，包括断电、错误和崩溃等情况下都具有原子性。(来源：维基百科)

        .. seealso::

             :term:`ACID` 

            “原子性（通过维基百科）<https://en.wikipedia.org/wiki/Atomicity_(database_systems)>`_

    一致性
        ACID模型中的四个组成部分之一，确保任何事务都将数据库从一个有效状态带到另一个有效状态。 写入数据库的任何数据都必须根据所有定义的规则有效，包括但不限于  :term:`constraints`  ，级联，触发器及其任何组合。(来源：维基百科)

        .. seealso::

             :term:`ACID` 

            “一致性（通过维基百科）<https://en.wikipedia.org/wiki/Consistency_(database_systems)>`_

    隔离性
    隔离的
    隔离
    隔离级别
        :term:`ACID` 模型的隔离属性确保并发执行的事务结果是如果事务串行执行将得到的结果。每个事务必须在完全隔离的情况下执行，即如果T1和T2并发执行，那么每个事务都应该保持独立。 (来源：维基百科)

        .. seealso::

             :term:`ACID` 

            “隔离（通过维基百科）<https://en.wikipedia.org/wiki/Isolation_(database_systems)>`_

             :term:`read uncommitted` 

             :term:`read committed` 

             :term:`repeatable read` 

             :term:`serializable` 

    可重复读
        四个数据库  :term:`隔离级别`  之一，可重复读包括所有  :term:` read committed`  隔离的隔离内容，并且还包括任何在事务期间读取的特定行从事务开始后保持不变的“非重复读数据”。这意味着，在事务期间，特定行不会更改，即使多个事务同时尝试更改原始行。(来源：维基百科)

    读已提交
        四个数据库  :term:`隔离级别`  之一，read committed提供的功能是：事务不会暴露给尚未提交的并发事务的任何数据，从而防止所谓的“dirty reads”。 但是，在read committed下可以有非重复读数据，这意味着当另一个事务提交更改时，行中的数据可能会更改通读取两次。

    读未提交
        四个数据库  :term:`隔离级别`  之一，read uncommitted的功能是：在事务提交之前，对数据库数据所做的更改不会成为永久更改。 然而，在read uncommitted状态下，可能可以在另一个事务的范围内查看未提交的数据；这些称为“dirty reads”。

    可串行化
        四种数据库  :term:`隔离级别`  之一，可串行化包括所有  :term:` repeatable read`  隔离中的所有隔离，并且在基于锁定的方法中，通过将行或一系列行锁定在锁定区间内可以保证所谓的“幻影读”不会发生，这意味着在此事务中读取的任何行都将继续存在，不存在的行将保证不能从另一个事务插入。

        可串行化隔离通常依赖于锁定行或行范围来实现此效果，并且可能会增加死锁的机会并降低性能。还存在一些基于非锁定的方案，但是这些方案必然依赖于拒绝事务，如果检测到写入冲突，则拒绝事务。

    耐久性
        耐久性是  :term:`ACID`  模型的一个属性，指一旦事务提交，即使在数据库在提交后立即崩溃时，它也将保持提交状态。 例如在关系数据库中，一组SQL语句执行后，结果需要永久储存。（来源：维基百科）

        .. seealso::
             :term:`ACID` 

            “耐久性（通过维基百科）<https://en.wikipedia.org/wiki/Durability_(database_systems)>`_

    RETURNING
        这是由某些后端提供的非SQL标准子句的各种形式，该子句提供了对执行INSERT、UPDATE或DELETE语句后返回结果集的服务。 任何匹配行中的一组列都可以返回，就像它们是从SELECT语句产生的结果集一样。

        RETURNING子句为常见的更新/选择方案提供了巨大的性能提升，包括在创建时检索内联或默认生成的主键值和默认值，以及以原子方式获取服务器生成的默认值。

        Postgresql的RETURNING常见使用示例如下:

        .. sourcecode:: sql

            INSERT INTO user_account (name) VALUES ('new name') RETURNING id, timestamp

        在上面的例子中，INSERT语句在执行后将生成功能 id 和 user_account.timestamp 的结果集，如上并未指定，因为它们是默认的值（但请注意，可以将任何系列的列或SQL表达式放入RETURNING中，而不仅仅是默认值列）。

        许多实现RETURNING或类似结构的有PostgreSQL、SQL Server、Oracle和Firebird。PostgreSQL和Firebird的实现通常非常齐全，而SQL Server和Oracle的实现具有警告。在SQL Server上，该子句称为“OUTPUT INSERTED”（针对INSERT和UPDATE语句）和“OUTPUT DELETED”（针对删除语句），关键警告是限制了触发器与此关键字结合使用。在Oracle上，它被称为“RETURNING INTO”，并且需要将该值放入OUT参数中，这意味着语法不仅笨拙，而且只能一次将其用于一行。

        SQLAlchemy的  :meth:`.UpdateBase.returning`  提供了在这些后端的RETURNING系统之上提供一层抽象的系统，以提供返回列的一致接口。 ORM还包含许多优化，可在可用时使用RETURNING。

    一对多
        一种 :func:`~sqlalchemy.orm.relationship` 样式，将父映射器的主键链接到相关表中的外键。然后，每个唯一的父对象可以引用零个或多个唯一的相关对象。

        相关对象反过来将具有隐含或显式的  :term:`多对一`  关系到它们的父对象。

        一个一对多架构示例（值得注意的是，它与：term:`多对一`架构相同）：

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

        从“department”到“employee”的关系是一对多，因为许多员工记录可以与单个部门相关联。 SQLAlchemy映射可能如下所示::

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

             :term:`多对一` 

             :term:`backref` 

    多对一
        一种 :func:`~sqlalchemy.orm.relationship` 样式，将父映射器中的外键链接到相关表的主键。然后，每个父对象可以引用正好零个或一个相关对象。

        相关对象反过来将对任意数量的引用对象有一个显式或隐含的  :term:`一对多`  关系。

        多对一架构示例（值得注意的是，它与：term:`一对多`架构相同）：

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


        从“employee”到“department”的关系是多对一，因为许多员工记录可以与单个部门相关联。 SQLAlchemy映射可能如下所示::

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

             :term:`一对多` 

             :term:`backref` 

    Backref
    双向关系
         :term:`relationship` ~sqlalchemy.orm.relationship` 对象相互关联，以便它们在内存中协作响应对任一侧的更改。这两个关系通过使用  :func:`~sqlalchemy.orm.relationship` ` backref``关键字，以便另一个 :func:`~sqlalchemy.orm.relationship` 被自动创建。我们可以针对我们在：term:`一对多`中使用的示例说明如下：

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

        可以将backref应用于任何关系，包括：term:`one to many`，  :term:`many to one`  和  :term:` many to many`  。

        .. seealso::

             :term:`relationship` 

             :term:`one to many` 

             :term:`many to one` 

             :term:`many to many` 

    多对多
        一种 :func:`sqlalchemy.orm.relationship` 样式，其中通过在中间使用中间表将两个表链接在一起。使用此配置，左侧的任意数量的行可能与右侧的任意数量的行相关，反之亦然。

        看一个将员工与项目相关联的模式：

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

        在上述中，“employee_project”表是多对多表，其自然形成由每个相关表的主键组成的复合主键。

        在SQLAlchemy中， :func:`sqlalchemy.orm.relationship` 函数可以在大多数情况下以大多数透明的方式表示此样式的关系，其中多对多表使用普通的表元数据指定：

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

        上述“Employee.projects”和反向引用“Project.employees”集合被定义为::

            proj = Project(name="Client A")

            emp1 = Employee(name="emp1")
            emp2 = Employee(name="emp2")

            proj.employees.extend([emp1, emp2])

        .. seealso::

             :term:`association relationship` 

             :term:`relationship` 

             :term:`one to many` 

             :term:`many to one` 

    关系
    关系
        连接两个映射类之间的连接单元，对应于数据库中两个表之间的某种关系。

        使用SQLAlchemy函数  :func:`~sqlalchemy.orm.relationship`  、  :term:` many to one`  或  :term:`many to many`  。 通过这种分类，关系构造处理根据当前的链接在数据库中当前链接的情况基于在内存中的对象协作响应对它们进行持久化。

        .. seealso::

              :ref:`relationship_config_toplevel` 

    游标
        一种控制结构，用于遍历数据库中的记录。在Python DBAPI中，光标对象实际上是语句执行的起点，也是用于获取结果的接口。

        .. seealso::

            “光标对象（在pep-249中）<https://www.python.org/dev/peps/pep-0249/#cursor-objects>`_

            “光标（通过维基百科）<https://en.wikipedia.org/wiki/Cursor_(databases)>`_employee_id = Column(Integer, ForeignKey("employee.id"), primary_key=True)
    project_id = Column(Integer, ForeignKey("project.id"), primary_key=True)
    role_name = Column(String(30))

    project = relationship("Project", backref="project_employees")
    employee = relationship("Employee", backref="employee_projects")

可以使用EmployeeProject来给项目添加员工::

    proj = Project(name="Client A")

    emp1 = Employee(name="emp1")
    emp2 = Employee(name="emp2")

    proj.project_employees.extend(
        [
            EmployeeProject(employee=emp1, role_name="技术主管"),
            EmployeeProject(employee=emp2, role_name="客户执行"),
        ]
    )

.. seealso::

     :term:`多对多` 

constraint
constraints
constrained
    关系型数据库中确保数据有效性和一致性的规则。 常见的约束形式包括:  :term:`主键约束` ,
     :term:`外键约束` .

candidate key

    :term:`关系代数` 术语，指标识行的唯一标识键属性或属性集。 一行可能有多个候选键，
    每个键都适用于该行的主键。表的主键始终是一个候选键。

    .. seealso::

         :term:`primary key` 

        `Candidate key (via Wikipedia) <https://en.wikipedia.org/wiki/Candidate_key>`_

        https://www.databasestar.com/database-keys/

primary key
primary key constraint

    在表中唯一定义每行特征的  :term:`约束` 。主键必须由任何其他行无法复制的特性组成。
    主键可以由单个属性或多个属性组合组成。 (via Wikipedia)

    表的主键通常在 "CREATE TABLE" 的  :term:`DDL`  中定义：

    .. sourcecode:: sql

        CREATE TABLE employee (
             emp_id INTEGER,
             emp_name VARCHAR(30),
             dep_id INTEGER,
             PRIMARY KEY (emp_id)
        )

    .. seealso::

         :term:`复合主键` 

        `Primary key (via Wikipedia) <https://en.wikipedia.org/wiki/Primary_Key>`_

composite primary key

    有两个或多个列的  :term:`主键` 。根据两个或多个列而不是单个值，可以唯一标识特定的数据库行。

    .. seealso::

         :term:`primary key` 

foreign key constraint

    两个表之间的引用约束。 父表上的可选键匹配另一张表的  :term:`候选键` ，构成所谓的ForeignKey。
    可以将外键用于交叉引用表。(via Wikipedia)

    可以使用以下标准SQL向表添加外键约束:

    .. sourcecode:: sql

        ALTER TABLE employee ADD CONSTRAINT dep_id_fk
        FOREIGN KEY (employee) REFERENCES department (dep_id)

    .. seealso::

        `Foreign Key Constraint (via Wikipedia)
        <https://en.wikipedia.org/wiki/Foreign_key_constraint>`_

check constraint

    在关系数据库的表中添加数据时，定义验证数据的条件的一种约束。 对于表中的每一行，都应用了检查约束。
    (via Wikipedia)

    可以使用以下标准SQL向表添加检查约束：

    .. sourcecode:: sql

        ALTER TABLE distributors ADD CONSTRAINT zipchk CHECK (char_length(zipcode) = 5);

    .. seealso::

         `CHECK constraint (via Wikipedia) <https://en.wikipedia.org/wiki/Check_constraint>`_

unique constraint
unique key index

    唯一键索引可唯一标识数据库表中每行数据值。 唯一键索引包含一个单独的列或一个单一数据库表中的列组合。
    如果没有使用NULL值，则在这些唯一键索引列中没有两行或数据记录可以拥有相同的数据值（或数据值组合）。
    根据其设计，数据库表可能具有许多唯一键索引，但最多只能有一个主键索引。

    (via Wikipedia)

    .. seealso::

        `Unique key (via Wikipedia) <https://en.wikipedia.org/wiki/Unique_key#Defining_unique_keys>`_

transient

    一个  :term:`Session`  中一个对象可以具有的主要对象状态，暂态对象是没有任何数据库标识并且还未与会话关联的新对象。
    将对象添加到会话后，它就会进入  :term:`pending`  状态。

    .. seealso::

          :ref:`session_object_states` 

pending

    一个  :term:`Session`  中一个对象可以具有的主要对象状态之一；暂挂对象是一个没有任何数据库标识但最近已与会话相关联的新对象。
    当会话发出 flush 并插入行时，对象会移动到  :term:`persistent`  状态。

    .. seealso::

          :ref:`session_object_states` 

deleted

    在 Session 中，一个对象可以具有的主要对象状态之一；一个已删除的对象是曾经是persistent并已在 flush 中向数据库发出 DELETE 语句以删除其行的对象。
    一旦会话的事务提交，对象就会移动到  :term:`detached`  状态；或者如果会话的事务被回滚，则删除会被恢复，该对象将返回到  :term:` persistent`  状态。

    .. seealso::

          :ref:`session_object_states` 

persistent

    在  :term:`Session`  中一个对象可以具有的主要对象状态之一；persistent 对象具有数据库标识(即主键) 并且当前与会话相关联。
    任何先前的  :term:`pending`  并已经插入的对象都处于 persistent 状态，会话从数据库中加载的任何对象也是，移除会话中的 persistent 对象后，就会成 detached 对象。

    .. seealso::

          :ref:`session_object_states` 

detached

    在  :term:`Session`  中一个对象可以具有的主要对象状态之一；一个 detached 对象具有数据库标识(即主键) 但未与任何会话相关联。
    因为它已从其会话中删除（无论是被删除还是所有权会话被关闭），所以曾经是 persistent 的对象进入到 detached状态。当在会话之间移动对象或将对象移动到 / 自外部对象缓存时，通常使用 detched 状态。

    .. seealso::

          :ref:`session_object_states` 

attached

    表示当前与特定  :term:`Session`  相关联的 ORM 对象。

    .. seealso::

          :ref:`session_object_states` 