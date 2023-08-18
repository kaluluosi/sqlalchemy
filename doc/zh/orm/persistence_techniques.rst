=================================
附加的持久化技术
=================================



.. _flush_embedded_sql_expressions:

将SQL插入/更新表达式嵌入刷新
================================================== ==

此功能允许将数据库列的值设置为SQL表达式而不是文字值。 它对于原子更新，调用存储过程等特别有用。 您只需将一个表达式分配给属性即可::

    class SomeClass(Base):
        __tablename__ = "some_table"

        # ...

        value = mapped_column(Integer)


    someobject = session.get(SomeClass, 5)

    #将 'value' 属性设置为 SQL 表达式 加上1
    someobject.value = SomeClass.value + 1

    #会产生 "UPDATE some_table SET value=value+1"
    session.commit()

此技术适用于INSERT和UPDATE语句。 刷新/提交操作后，上面的“someobject”的“value”属性将过期，因此下次访问时新生成的值将从数据库加载。

该功能还具有条件支持，以与主键列一起使用。 对于支持 RETURNING
的后端（包括Oracle，SQL Server，MariaDB 10.5，SQLite 3.35），也可以将SQL表达式分配给主键列。 这既允许评估SQL表达式，又允许成功检索对INSERT修改主键值的任何服务器端触发器作为对象的主键的一部分：

    class Foo(Base):
        __tablename__ = "foo"
        pk = mapped_column(Integer, primary_key=True)
        bar = mapped_column(Integer)


    e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
    Base.metadata.create_all(e)

    session = Session(e)

    foo = Foo(pk=sql.select(sql.func.coalesce(sql.func.max(Foo.pk) + 1, 1)))
    session.add(foo)
    session.commit()

在PostgreSQL上，上面的:class：`Session`将发出以下INSERT：

.. sourcecode:: sql

    INSERT INTO foo (foopk, bar) VALUES
    ((SELECT coalesce(max(foo.foopk) + %(max_1)s, %(coalesce_2)s) AS coalesce_1
    FROM foo), %(bar)s) RETURNING foo.foopk

.. versionadded:: 1.3
    ORM刷新期间现在可以通过主键列传递SQL表达式； 如果数据库支持RETURNING或如果正在使用pysqlite，则ORM将能够将服务器生成的值作为主键属性的值检索。

.. _session_sql_expressions:

在会话中使用SQL表达式
====================================

SQL表达式和字符串可以通过
:class:`~sqlalchemy.orm.session.Session`在其事务上下文中执行。
这通常使用:meth:`~.Session.execute`方法最容易完成，该方法返回
:class:`~sqlalchemy.engine.CursorResult`，以与
:class:`~sqlalchemy.engine.Engine`或
:class:`~sqlalchemy.engine.Connection`相同的方式::

    Session = sessionmaker(bind=engine)
    session = Session()

    #执行字符串语句
    result = session.execute("select * from table where id=:id", {"id": 7})

    #执行SQL表达式结构
    result = session.execute(select(mytable).where(mytable.c.id == 7))

当前的:paramref：`~ SQLAlchemy.engine.Connection`由持有
:meth:`~.Session.connection`方法访问::

    connection = session.connection()

上述示例处理绑定到单个类的:class:`_engine.Engine`或
:class:`_engine.Connection`的:class:`_orm.Session`。要执行使用多个
引擎或根本不绑定（即依赖于已绑定元数据）的：class:`_orm.Session“语句，
:meth:`_orm.Session.execute`和
:meth:`_orm.Session.connection`接受绑定参数的字典
:paramref:`_orm.Session.execute.bind_arguments`，其中可能包括“mapper”
传递映射类或：class:`_orm.Mapper`实例，其中用于定位所需引擎的适当上下文::

    Session = sessionmaker()
    session = Session()

    #在执行时需要指定映射器或类
    result = session.execute(
        text("select * from table where id=:id"),
        {"id": 7},
        bind_arguments={"mapper": MyMappedClass},
    )

    result = session.execute(
        select(mytable).where(mytable.c.id == 7), bind_arguments={"mapper": MyMappedClass}
    )

    connection = session.connection(MyMappedClass)

.. versionchanged:: 1.4 :meth:`_orm.Session.execute`和
   现在将“映射器”和“子句”参数作为字典的一部分传递，
    发送名为:paramref:`_orm.Session.execute.bind_arguments`的初始值。
   先前的参数仍然可接受，但是此用法已被弃用。

.. _session_forcing_null:

对具有默认值的列强制使用NULL
==========================================

ORM将任何未在对象上显式设置的属性视为“默认值”情况；该属性将在INSERT语句中被省略：

    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String(50), nullable=True)


    obj = MyObject(id=1)
    session.add(obj)
    session.commit()  #带有'data'列省略的INSERT;数据库
                    #本身将它保存为NULL值

从INSERT中省略列意味着该列将具有设置为NULL的值，*除非*该列具有默认设置，在这种情况下，将保留默认值。这适用于SQL的纯视角，其中包括具有服务器端默认值的服务器端，以及在具有客户端和服务器端默认值的情况下使用SQLAlchemy的插入行为：

    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String(50), nullable=True, server_default="default")


    obj = MyObject(id=1)
    session.add(obj)
    session.commit()  #INSERT，其中 ‘ data '列省略;数据库
                    #本身将它保留为值'default'

但是，对于ORM，即使在对象上显式将Python值None分配给该对象，这也被视为与值未分配同​​​​​​样::

    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String(50), nullable=True, server_default="default")


    obj = MyObject(id=1, data=None)
    session.add(obj)
    session.commit()  #INSERT，其中 'data'列显式设置为None;
                      #ORM仍会将其从语句中省略，而且
                    #数据库仍将其保存为值'default'

上述操作将在“data”列中保留服务器默认值“default”，而不是SQL NULL，即使传递了“None”也是ORM的长期行为，许多应用程序将其视为一种假设。

那么，如果我们希望实际将NULL放入此列中，即使该列具有默认值，怎么办？ 有两种方法。 其一是在每个实例级别上使用:obj:`_expression.null` SQL构造分配属性::

    from sqlalchemy import null

    obj = MyObject(id=1, data=null())
    session.add(obj)
    session.commit()  #在'data'列上显式设置为‘null（）’;ORM直接使用它，绕过所有客户端-和服务器端默认值，数据库将将其保留为NULL值

: obj:`_expression.null` SQL构造将始终转换为SQL NULL值直接出现在目标INSERT语句中。

如果我们希望能够使用Python值“None”，并且尽管存在列默认值，此值也将作为NULL持久化，我们可以使用Core级别的修改器
:meth:`.TypeEngine.evaluates_none`，指示ORM该如何将值“None”视为其他任何值并传递它，而不是将其视为“缺少”值，例如该。

    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(
            String(50).evaluates_none(),  #指示None始终应传递
            可识别，可为空，server_default=“default”。
        )


    obj = MyObject(id=1，data = None)
    session.add(obj)
    session.commit()  #将“data”列显式设置为None;
                      #ORM将直接使用它，绕过所有客户端-
                    #和服务器端默认值，数据库将
                    #将其保留为NULL值

.. topic::评估无

  :meth:`.TypeEngine.evaluates_none`修改器主要用于表示值Python的类型中
  “无（None）”的意义，最基本的示例是JSON类型，旨在保留JSON的应用程序
  ``null```值而不是SQL NULL。 我们在这里稍微重塑它来
  表示ORM应该在出现时传递'None'值，并且尽管未分配任何特殊类型级别的行为，仍然传递它。

.. _orm_server_defaults:

获取服务器生成的默认值
================================================== ==

如前所述，Core支持有关数据库列的概念，对于这些列，数据库本身在进行INSERT和在较少的情况下进行UPDATE语句时会生成一个值。 ORM特征支持此类列，以便在刷新时能够获取这些新生成的值。 如果主键列生成服务器，则此行为对于必须知道对象的主键值一旦它被持久化的情况是必需的。

在绝大多数情况下，具有其值由数据库自动生成的主键列仅为简单必须列，这些列由数据库实现为所谓的“自动递增”列，或者与该列相关联的序列。 SQLALCHEMY都支持这些类型，数据库包括获取“最后插入的ID”之类的功能，其中在不支持RETURNING并且RETURNING不受支持时将自动生成值。

对于不是主键列或不是简单的自动递增整数列的服务器生成列，ORM要求将这些列标记为适当的“服务器默认值”指令，以允许ORM检索此值。 然而，并非所有的方法都支持所有后端，因此必须注意使用适当的方法。 需要回答的两个问题是，1.此列是否属于主键或不是，
此外，该数据库是否支持RETURNING或等效项，例如“OUTPUT inserted”。这些是返回服务器生成值的SQL短语，在调用INSERT或UPDATE语句的同时返回该值。目前，POSTGRES，ORACLE，MariaDB 10.5，SQLite 3.35和SQL Server都支持RETURNING。

情况1：非主键，支持RETURNING或等效项

在此情况下，列应标记为：class：'FetchedValue'或具有显式值的：paramref：'_schema.Column.server_default'。 如果:paramref：'_orm.Mapper.eager_defaults'参数设置为`True'或对于支持两者的dialect默认设置为`“auto”`，则ORM将在执行INSERT语句时自动将这些列添加到RETURNING子句中，如：一样：

    class MyModel(Base):
        __tablename__ = "my_table"

        id = mapped_column(Integer, primary_key=True)

        # 服务器端SQL日期函数生成新时间戳
        timestamp = mapped_column(DateTime(), server_default=func.now())

        # 其他非此处命名的服务器端函数，例如触发器，
        # 。在INSERT期间将值填充到此列中
        special_identifier = mapped_column(String(50), server_default=FetchedValue())

        #将急切的默认值设置为True。这通常是可选的，因为如果
        #后端支持RETURNING + insertmanyvalues，急切默认值将
        #对于在INSERT时添加新生成的值立即起效

上面，没有从客户端端为“timestamp”或“special_identifier”指定明确值的INSERT语句将在RETURNING子句中包含“timestamp”和“special_identifier”列，以便立即可用。 在PostgreSQL数据库上，上面的INSERT将如下所示：

.. sourcecode:: sql

   INSERT INTO my_table（）VALUES（（SELECT coalesce（max（my_table.id）+％（max_1）s，％（coalesce_2）s）作为coalesce_1
   从my_table），％（id_1）s）返回my_table.id，my_table.timestamp，my_table.special_identifier

.. versionadded:: 2.0.0rc1 :paramref:`_orm.Mapper.eager_defaults`参数现在为新设置
   一个名为 `“auto”` 的设置，如果支持，并让映射：class：'“ORM将能够使用RETURNING检索使用聚合值”`。'insertmanyvalues'，急切默认值将在默认情况下生效。

.. note:: :paramref:`_orm.Mapper.eager_defaults`的值'“auto”'仅
  适用于INSERT语句。除非：paramref:`_orm.Mapper.eager_defaults`设置为
  “True”，否则UPDATE语句将不使用RETURNING，即使可用。
  这是因为没有相应的“insertmanyvalues”功能可用于UPDATE，因此UPDATE RETURNING将要求单独发出UPDATE语句
  针对每个正在UPDATE的行。

情况2：表包括与RETURNING不兼容的触发器生成的值
------------------------------------------------------------

'auto' :paramref：'_orm.Mapper.eager_defaults'参数的设置意味着
支持RETURNING的后端通常会使用RETURNING在INSERT语句中立即获取新生成的默认值。 但是，触发器生成值的限制如下，以致不能使用RETURNING：

    • SQL Server不允许在INSERT语句中使用RETURNING检索触发器生成的值；该语句将失败。

    • SQLite在将RETURNING与触发器组合使用时存在限制，因此
     RETURNING子句不会将INSERTed值可用

    • 其他后端可能具有与触发器一起支持RETURNING的限制，或与之相关的其他类型的服务器生成值。

要禁用用于此类值的RETURNING，包括不仅限于服务器生成的默认值，还要确保映射的：class:`.Table`指定：paramref:`_schema.Table.implicit_returning`为“False”。使用Declarative映射，这看起来像：

    class MyModel(Base):
        __tablename__ ='my_table'

        id: Mapped[int] = mapped_column(primary_key=True)
        data: Mapped[str] = mapped_column(String(50))

        #假设数据库触发器在INSERT期间将值填充到此列中
        special_identifier = mapped_column(String(50)， server_default=FetchedValue())

        #禁用表的所有RETURNING用法
        __table_args__ = {'implicit_returning'：False}

在使用pyodbc驱动程序的SQL Server上，上面表的INSERT将不使用RETURNING并使用SQL Server“scope_identity()”函数来检索新生成的主键值：

.. sourcecode:: sql

    INSERT INTO my_table（data）VALUES（？），选择作用范围_identity（）

.. seealso::

    :ref：`mssql_insert_behavior`– SQL Server dialect的背景
    获取新生成的主键值的方法

情况3：非主键，RETURNING或等效项不被支持或不需要

在这种情况下，通常不想使用:paramref：`.orm.Mapper.eager_defaults`，因为在缺少RETURNING支持的情况下，其当前实现是发出选择一行的SELECT-for-row， 不可行的。 因此，在映射下省略参数：paramref：'.orm.Mapper.eager_defaults'：

    class MyModel(Base):
        __tablename__ ='my_table'

        id = mapped_column(Integer, primary_key=True)
        timestamp = mapped_column(DateTime()， server_default=func.now())

        #假设数据库触发器在INSERT期间将值填充到此列中
        special_identifier = mapped_column(String(50)， server_default=FetchedValue())

在在不包括RETURNING或“ `insertmanyvalues'支持的后端处理上述映射作为新的插入的记录后，“timestamp”和“special_identifier”列将保持为空，并且在刷新后第一次访问时将通过第二个SELECT语句进行抓取，例如他们被标记为“过期”。

如果提供了明确值为`True'的:paramref：`.orm.Mapper.eager_defaults'，并且后端数据库不支持RETURNING或等效项，则ORM将紧随INSERT语句后立即发出SELECT语句以获取新生成的值。每个INSERT行生成一个SELECT语句。这通常不理想，因为它会向冲洗过程中添加可能不需要的附加SELECT语句。使用上面的映射和:paramref：`.orm.Mapper.eager_defaults’标志的版本针对MySQL（不是MariaDB）会产生如下SQL：

.. sourcecode:: sql

    INSERT INTO my_table（）VALUES（（））
    当急切的默认值使用时，但RETURNING不受支持时
    SELECT my_table.timestamp作为my_table_timestamp，my_table.special_identifier作为my_table_special_identifier
    从my_table中WHERE my_table.id =％s

SQLALCHEMY的以后的版本可以尝试通过一次语句SELECT许多新插入的行来提高急性默认值在缺少RETURNING的情况下的效率。__mapper_args__ = {"eager_defaults": True}

具有类似上面的映射后，ORM用于INSERT和UPDATE的SQL将在RETURNING子句中包括“created”和“updated”：

.. sourcecode:: sql

  INSERT INTO my_table (created) VALUES (now()) RETURNING my_table.id, my_table.created, my_table.updated

  UPDATE my_table SET updated=now() WHERE my_table.id = %(my_table_id)s RETURNING my_table.updated



.. _orm_dml_returning_objects:


使用INSERT，UPDATE和ON CONFLICT（即upsert）返回ORM对象
==========================================================================

SQLAlchemy 2.0包括增强的功能，用于发射几个种类的ORM启用的INSERT、UPDATE和upsert语句。参见文档： :doc:`queryguide/dml` 以供文档。有关upsert，请参见：:ref:`orm_queryguide_upsert`。

使用带RETURNING的PostgreSQL ON CONFLICT返回upserted ORM的对象
---------------------------------------------------------------------------

该部分已移至：:ref:`orm_queryguide_upsert`。

.. _session_partitioning:

分区策略（例如，每个Session的多个数据库后端）
=====================================================================

简单垂直分区
----------------------------

竖直分区将不同的类、类层次结构或映射表配置到多个数据库中，
通过在:class:`.Session`中配置:paramref:`.Session.binds`参数实现。
该参数接收一个字典，其中包含任何组合ORM-mapped类、映射层次结构内的任意类
（例如，声明基类或mixin）、:class:`_schema.Table`对象和
:class:`_orm.Mapper`对象作为键，这些键通常指向:class:`_engine.Engine`或
不太典型的：class:`_engine.Connection`对象一样的目标。
每当:class:`.Session`需要代表特定类型的映射类发射SQL以定位适当的数据库连接源时，
将查找该字典：

    engine1 = create_engine("postgresql+psycopg2://db1")
    engine2 = create_engine("postgresql+psycopg2://db2")

    Session = sessionmaker()

    # 将 User 操作绑定到 engine1，将 Account 操作绑定到 engine2
    Session.configure(binds={User: engine1, Account: engine2})

    session = Session()

上述代码中，任何类的SQL操作都将使用其链接到的引擎 class:`_engine.Engine`。 该功能在读写操作方面是全面的；针对映射到“engine1”的实体的:class:`_query.Query`（通过查看所请求的项目列表中的第一个实体确定）将使用“engine1”来运行查询。flush操作将对每个类，即“User”和“Account”，基于每个类使用**两个**引擎。

在更常见的情况下，通常有基础类或mixin类可以用来区分即将用于不同数据库连接的操作。 ：paramref:`.Session.binds`参数可以容纳任何任意Python类作为键，如果它被发现在特定映射类的__mro__（Python方法解析顺序）中，将使用该键。假设两个声明基类分别表示两种不同的数据库连接：

    from sqlalchemy.orm import DeclarativeBase, Session

    class BaseA(DeclarativeBase):
        pass

    class BaseB(DeclarativeBase):
        pass

    class User(BaseA):
        ...

    class Address(BaseA):
        ...

    class GameInfo(BaseB):
        ...

    class GameStats(BaseB):
        ...

    Session = sessionmaker()

    # 所有 User / Address 操作都将在 engine1 上执行，所有
    # Game 操作将在 engine2 上执行
    Session.configure(binds={BaseA: engine1, BaseB: engine2})

上述代码中，从"BaseA"和"BaseB"派生的类的SQL操作将根据它们是从哪个超类按继承顺序继承的，选择将其路由到两个引擎之一。

.. seealso::

    ：paramref:`.Session.binds`

多引擎Session的交易协调
----------------------------------------------------------

使用多个绑定引擎的一个警告是，当在一个引擎上提交操作成功后，在另一个引擎上提交操作可能会失败，这是一个不一致问题，在关系数据库中通过“两阶段事务”解决，它添加了一个附加的“准备”步骤，使多个数据库在实际完成事务之前同意提交。

由于DBAPI的支持有限，SQLAlchemy对跨后端的两阶段事务提供有限支持。最常见的是，它已被证明在PostgreSQL后端中可行，并在MySQL后端中实现了较小的影响。然而，:class:`.Session`完全能够利用两阶段事务特性，当后端支持时，通过设置:class:`.sessionmaker`或：class：`。Session`中的：paramref:`.Session.use_twophase`标志。参见：ref：`two phase transaction<session_twophase>`，以获取示例。

.. _session_custom_partitioning:

自定义垂直分区
----------------------------

可以通过覆盖:meth:`.Session.get_bind`方法来构建更全面的基于规则的类级分区。下面我们演示一个自定义:class:`.Session`，其中提供以下规则:
1. flush操作以及批量的"更新"和"删除"操作均传递给名为'leader'的引擎。
2. 子类化“myotherclass”的对象的操作均在“other”引擎上发生。
3. 所有其他类的读操作都在“follower1”或“follower2”的数据库上随机选择。

    engines = {
        "leader": create_engine("sqlite:///leader.db"),
        "other": create_engine("sqlite:///other.db"),
        "follower1": create_engine("sqlite:///follower1.db"),
        "follower2": create_engine("sqlite:///follower2.db"),
    }

    from sqlalchemy.sql import Update, Delete
    from sqlalchemy.orm import Session, sessionmaker
    import random

    class RoutingSession(Session):
        def get_bind(self, mapper=None, clause=None):
            if mapper and issubclass(mapper.class_, MyOtherClass):
                return engines["other"]
            elif self._flushing or isinstance(clause, (Update, Delete)):
                # NOTE: this is for example, however in practice reader/writer splits are likely more straightforward by using two distinct
                # Sessions at the top of a "reader" or "writer" operation.
                # See note below
                return engines["leader"]
            else:
                return engines[random.choice(["follower1", "follower2"])]

上述:class:`。Session`类使用"class_"参数插入到:class:`. sessionmaker`中：

    Session = sessionmaker(class_=RoutingSession)

此方法可以与多个:class:`_schema.MetaData`对象相结合，使用诸如使用
声明性的“__abstract__”关键字等方法，如:ref:`declarative_abstract`所述。

.. note:: 尽管上面的示例说明了基于“leader”或“follower”数据库的特定SQL语句的路由操作，但这可能是不切实际的方法，因为它导致了相同操作内读取和写入之间的不协调的事务行为。实际上，最好提前构造一个“reader”或“writer”会话的:class:`_orm.Session`，根据正在进行的总体操作/事务，基于整个操作/事务设置一个操作将使用的类:`_orm.sessionmaker`的场景。这样，将写入数据的操作也将在同一个事务范围内发射其读取查询。参见:ref:`session_transaction_isolation_enginewide`中设置一个:class:`_orm.sessionmaker`的示例，以进行仅“只读”操作的会话，并包含DML/COMMIT的另一个会话。

.. seealso::

    “SQLAlchemy中的Django-style数据库路由器<https://techspotzzzeek。org/2012/01/11/django-style-database-routers-in-sqlalchemy/>” - 博客，更详细地说明了:meth:`。Session.get_bind`的示例。

水平分区
-----------------------

水平分区将单个表的行（或一组表）分布到多个数据库中。 SQLAlchemy :class:`.Session`包含了这个概念的支持，但要完全使用它需要使用:class:`.Session`和:class:`_query.Query`子类。这些子类的基本版本在:ref:`horizontal_sharding_toplevel` ORM扩展中可用。使用示例在：：`：ref：`水平分片示例`。

.. _bulk_operations:

批量操作
===============

.. legacy::

  SQLAlchemy 2.0将“ORM插入”和“ORM更新”的批量插入和批量更新功能集成到2.0风格的:meth:`_orm.Session.execute`方法中，直接使用:class:`_dml.Insert`和:class:`_dml.Update`结构。有关文档，请参见：: doc: `queryguide/dml`，包括：ref:`orm_queryguide_legacy_bulk_insert`，它说明从旧方法迁移到新方法的方法。