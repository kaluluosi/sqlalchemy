附加的持久化技术
===================

.. _flush_embedded_sql_expressions:

将SQL插入/更新表达式嵌入到Flush中
===================================

此功能允许将数据库列的值设置为SQL表达式而不是文字值。它特别适用于原子更新、调用存储过程等。你只需要将表达式分配给属性即可::

    class SomeClass(Base):
        __tablename__ = "some_table"

        # ...

        value = mapped_column(Integer)


    someobject = session.get(SomeClass, 5)

    # 将"value"属性设置为SQL表达式加1
    someobject.value = SomeClass.value + 1

    # 发布 "UPDATE some_table SET value=value+1"
    session.commit()

此方法适用于INSERT和UPDATE语句。在Flush/commit操作之后，“someobject”上的“value”属性过期，因此在下次访问时，新生成的值将从数据库中加载。

该功能还具有条件支持，以与主键列一起使用。对于具有RETURNING支持的后端(包括Oracle、SQL Server、MariaDB 10.5、SQLite 3.35)，SQL表达式还可以分配到主键列中。这样，即可以计算SQL表达式，也可以成功检索单个对象的主键中修改主键值的任何服务器端触发器::

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

在PostgreSQL上，上述 :class:`.Session` 会发出以下INSERT:

.. sourcecode:: sql

    INSERT INTO foo (foopk, bar) VALUES
    ((SELECT coalesce(max(foo.foopk) + %(max_1)s, %(coalesce_2)s) AS coalesce_1
    FROM foo), %(bar)s) RETURNING foo.foopk

.. versionadded:: 1.3
    SQL表达式现在可以在ORM中的Flush期间传递到主键列；如果数据库支持RETURNING，或者正在使用pysqlite，则ORM将能够检索服务器生成的值作为主键属性的值。

.. _session_sql_expressions:

在Sessions中使用SQL表达式
=======================

SQL表达式和字符串可以通过  :class:`~sqlalchemy.orm.session.Session` ~.Session.execute` 方法完成，该方法以与 :class:` ~sqlalchemy.engine.Engine`或
  :class:`~sqlalchemy.engine.Connection` ~sqlalchemy.engine.CursorResult` ::

    Session = sessionmaker(bind=engine)
    session = Session()

    # 执行一个字符串语句
    result = session.execute("select * from table where id=:id", {"id": 7})

    # 执行SQL表达式构造
    result = session.execute(select(mytable).where(mytable.c.id == 7))

  :class:`~sqlalchemy.orm.session.Session` ~sqlalchemy.engine.Connection` 可通过  :meth:`~.Session.connection`  方法获得::

    connection = session.connection()

上述示例处理绑定到单个  :class:`_engine.Engine` 。如果要使用绑定到多个引擎或未绑定任何引擎的 :class:` _orm.Session`执行语句(即依赖于绑定的元数据)，
  :meth:`_orm.Session.execute`  和  :meth:` _orm.Session.connection`  接受一个绑定参数字典，其中可能包括“mapper”参数，
这表示映射的类或 :class:`_orm.Mapper` 实例，用于定位所需引擎的正确上下文::

    Session = sessionmaker()
    session = Session()

    # 在执行时需要指定映射器或类
    result = session.execute(
        text("select * from table where id=:id"),
        {"id": 7},
        bind_arguments={"mapper": MyMappedClass},
    )

    result = session.execute(
        select(mytable).where(mytable.c.id == 7), bind_arguments={"mapper": MyMappedClass}
    )

    connection = session.connection(MyMappedClass)

.. versionchanged:: 1.4
    :meth:`_orm.Session.execute` ` mapper``和``clause``参数现在作为字典的一部分通过发送给  :paramref:`_orm.Session.execute.bind_arguments`  参数而传递。
   先前的参数仍然被接受，但是该用法已被弃用。

.. _session_forcing_null:

在具有默认值的列上强制执行NULL
=================================

ORM将任何未在对象上显式设置的属性都视为"default"情况；属性将从INSERT语句中省略::

    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String(50), nullable=True)


    obj = MyObject(id=1)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column omitted; the database
    # itself will persist this as the NULL value

从INSERT中省略列意味着该列将设置为NULL值，除非该列设置了默认值，在这种情况下，将保留默认值。从一个纯SQL的角度、从客户端和服务器端的默认行为以及使用客户端和服务器端默认的SQLAlchemy的插入行为都是如此：

　　
    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String(50), nullable=True, server_default="default")


    obj = MyObject(id=1)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column omitted; the database
    # itself will persist this as the value 'default'

但是，在ORM中，即使在对象上将Python值``None``显式分配给属性，这也被视为与未分配该值相同::

    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(String(50), nullable=True, server_default="default")


    obj = MyObject(id=1, data=None)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column explicitly set to None;
    # the ORM still omits it from the statement and the
    # database will still persist this as the value 'default'

上述操作将在``data``列中保留服务器的默认值``"default"``，而不是SQL NULL，即使传递了``None``；这是ORM的长期行为，许多应用程序将其视为假设。

那么，如果我们希望实际将NULL放入此列中，即使该列具有默认值，该怎么办？有两种方法。一种是在单个实例级别上使用  :obj:`_expression.null`   SQL构造分配属性::

    from sqlalchemy import null

    obj = MyObject(id=1, data=null())
    session.add(obj)
    session.commit()  # INSERT with the 'data' column explicitly set as null();
    # ORM直接使用它，绕过所有客户端和服务器
    # 默认值，数据库将将其持久化为NULL值

  :obj:`_expression.null`   SQL构造总是转换为目标INSERT语句中直接存在的SQL NULL值。

如果我们希望能够使用Python值``None``，并且这也被持久化为NULL，尽管存在列默认值，我们可以使用Core级修饰符  :meth:`.TypeEngine.evaluates_none`  进行ORM配置。该修饰符指示ORM要将“None”值与任何其他值相同地对待并将其传递，而不是将其视为缺少的“missing”值::

    class MyObject(Base):
        __tablename__ = "my_table"
        id = mapped_column(Integer, primary_key=True)
        data = mapped_column(
            String(50).evaluates_none(),  # 表明None应始终被传递
            nullable=True,
            server_default="default",
        )


    obj = MyObject(id=1, data=None)
    session.add(obj)
    session.commit()  # INSERT with the 'data' column explicitly set to None;
    # ORM直接使用它，绕过所有客户端和服务器
    # 默认值，数据库将将其持久化为NULL值

.. topic:: 根据None评估

   :meth:`.TypeEngine.evaluates_none`  修饰符主要用于表明Python值“None”是有意义的类型，主要示例是可以将JSON类型持久化为JSON ` `null``而不是SQL NULL。
  我们稍微重新解释它的作用，以便ORM能够在存在它时在类型中传递“None”，即使未分配特殊类型级行为。

.. _orm_server_defaults:

提取服务器生成的默认值
==========================

如   :ref:`server_defaults`  和   :ref:` triggered_columns`  中介绍的那样，Core支持数据库列，其中数据库本身在INSERT和在不太常见的情况下在UPDATE语句中生成一个值。ORM特征支持对这些列进行的查看，
以便在Flush时检索这些新生成的值。需要在具有服务器生成值的主键列的情况下进行此操作，因为ORM需要知道对象的主键一旦持久化就可以使用。

在绝大多数情况下，具有自动由数据库生成的主键列都是简单的整数列，这些列通过称为所谓的“自动增量”列的数据库实现，或者从与该列关联的序列中实现。SQLAlchemy Core中的每个数据库方言都支持检索这些主键值的方法，这通常是Python DBAPI原生支持的，通常情况下自动处理这个过程。 更多文档详见  :paramref:`_schema.Column.autoincrement`  。

对于不是主键列或不是简单自动增量整数列的服务器生成列，ORM要求将这些列标记为适当的``server_default``指令，以允许ORM检索此值。但是，并非所有方法都在所有后端上得到支持，因此必须小心地使用适当的方法。需要回答的两个问题是，1.此列是否是主键列，2.数据库是否支持RETURNING或等效内容，如"OUTPUT inserted" ; 这些是在调用INSERT或UPDATE语句的同时返回服务器生成值的SQL短语。RETURNING目前由PostgreSQL、Oracle、MariaDB 10.5、SQLite 3.35和SQL Server支持。

情况1：非主键，支持RETURNING或等效内容

在此情况下，列应使用  :class:`.FetchedValue`  或显式使用  :paramref:` _schema.Column.server_default`   标记。在Flush语句执行INSERT操作时，假设  :param:`_orm.Mapper.eager_defaults`  参数设置为 ` `True``或者对于支持 RETURNING
以及   :ref:`insertmanyvalues <engine_insertmanyvalues>`  包括在内的方言，ORM将自动将这些列添加到RETURNING子句中::

    class MyModel(Base):
        __tablename__ = "my_table"

        id = mapped_column(Integer, primary_key=True)

        # 服务器端SQL日期函数生成新的时间戳
        timestamp = mapped_column(DateTime(), server_default=func.now())

        # 在INSERT期间，数据库中的某些其他服务器端函数或触发器名称不在此处，会将一个值分配给该列
        special_identifier = mapped_column(String(50), server_default=FetchedValue())

        # 设置eager_defaults为True。这通常是可选的，因为如果
        # 后端支持RETURNING + insertmanyvalues，则INSERT会自动处理提示
        # 发生
        __mapper_args__ = {"eager_defaults": True}

上面，在客户端上未指定“timestamp”或“special_identifier”的值的INSERT语句将包括在RETURNING子句中的“timestamp”和“特殊标识符”，因此它们将立即可用。在PostgreSQL数据库上，上述表的INSERT如下所示：

.. sourcecode:: sql

   INSERT INTO my_table DEFAULT VALUES RETURNING my_table.id, my_table.timestamp, my_table.special_identifier

.. versionchanged:: 2.0.0rc1  :paramref:`_orm.Mapper.eager_defaults`  现在的值默认为一个新的设置"auto"，如果后端数据库既支持 RETURNING 又支持   :ref:` insertmanyvalues <engine_insertmanyvalues>` ，将自动使用 RETURNING 来获取插入的默认值。

.. note::  :paramref:`_orm.Mapper.eager_defaults`  的` `"auto"`值仅适用于INSERT语句。即使是可用的，UPDATE语句也不会使用RETURNING ，除非将  :paramref:`_orm.Mapper.eager_defaults`  设置为` `True``。这是因为不存在等效的 "insertmanyvalues" 功能，因此UPDATE RETURNING将需要为每个要UPDATE的行单独发出UPDATE语句。

情况2：表包含与RETURNING不兼容的触发器生成的值
------------------------------------------------------------

``"auto"``设置的  :paramref:`_orm.Mapper.eager_defaults`   意味着后端支持RETURNING时，常规情况下会使用RETURNING来获取新生成的默认值。然而，服务器生成值存在以下限制，例如使用触发器时RETURNING不能使用：

* SQL Server不允许在INSERT语句中使用RETURNING检索触发器生成的值；语句将失败。

* SQLite在将返回的值与触发器结合使用时存在限制，因此返回的子句将无法使用插入的值。

* 其他后端可能存在限制，无法与触发器或其他类型的服务器生成值一起使用RETURNING。

为了禁用这些值的RETURNING使用，这些值包含了但不仅限于服务器生成的默认值，以确保ORM能够检索到它们,  :paramref:`_schema.Table.implicit_returning` .Table` 的值为 ``False``。此类Declarative映射如下所示：

　
    class MyModel(Base):
        __tablename__ = "my_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        data: Mapped[str] = mapped_column(String(50))

        #假设一个触发器在INSERT期间将一个值分配给该列
        special_identifier = mapped_column(String(50), server_default=FetchedValue())

        #禁用表的所有RETURNING用法
        __table_args__ = {"implicit_returning": False}

在使用pyodbc驱动程序的SQL Server上，以上表的INSERT将不使用RETURNING，并使用SQL Server的``scope_identity()``函数检索新生成的主键值：

.. sourcecode:: sql

    INSERT INTO my_table (data) VALUES (?); select scope_identity()

.. seealso::

      :ref:`mssql_insert_behavior`  - SQL Server方言获取新增生成的主键值的背景

情况3：非主键，不支持或不需要RETURNING或等效
------------------------------------------------------------

此情况与上面的情况1相同，但通常不希望使用  :param:`_orm.Mapper.eager_defaults`  ，因为在没有RETURNING支持的情况下，它的当前实现是发出一个SELECT-per-row，这不是高效的。因此，在映射以下情况时略去参数::

    class MyModel(Base):
        __tablename__ = "my_table"

        id = mapped_column(Integer, primary_key=True)
        timestamp = mapped_column(DateTime(), server_default=func.now())

        # 假设数据库中的某些其他服务器端函数或触发器名称不在此处，将一个值分配给该列的INSERT
        special_identifier = mapped_column(String(50), server_default=FetchedValue())

在使用不包括RETURNING或“insertmanyvalues”支持的后端讯息时，上面的记录在执行Flush后加入中的“timestamp”和“special_identifier”列将保持为空，且在在Flush后首次访问它们时将通过第二个SELECT语句中的“过时”标记它们，例如它们在FLUSH之后。如果显式提供  :param:`_orm.Mapper.eager_defaults`   值 "True"，并且后端数据库不支持 RETURNING 或等效内容，
ORM将立即在 INSERT 语句之后发出一个SELECT语句，以获取新生成的值; ORM目前无法在没有RETURNING的情况下批量选择许多新插入的行。这通常是不希望的，因为它会在Flush过程中增加了额外的SELECT语句，可能不需要。在使用上面的映射与MySQL(而不是MariaDB)进行Flush时，会导致文本如下的SQL：

.. sourcecode:: sql

    INSERT INTO my_table () VALUES ()

    -- 当 eager_defaults **被**使用，但不支持 RETURNING
    SELECT my_table.timestamp AS my_table_timestamp, my_table.special_identifier AS my_table_special_identifier
    FROM my_table WHERE my_table.id = %s

在没有RETURNING的情况下，为了从服务器获取默认值，还要注意客户端生成的SQL表达式的问题。有关这些表达式的详细信息，可参考  :ref:`defaults_client_invoked_sql` 。这些SQL表达式目前在ORM中受到与真实服务器端默认值相同的限制；即使这些表达式不是DDL服务器端默认值并且由SQLAlchemy本身活跃渲染，它们也不会在  :param:` _orm.Mapper.eager_defaults`  设置为 ``"auto"``或``True``且没有与该列关联的  :class:`.FetchedValue`  上使用 :class:` .FetchedValue`指令来强制使用查看, 请参见下面的示例。


.. _orm_server_default_fetching_example:

示例
-----------------

以下是一个包含举例的示例，展示了两列被指定为数据库在INSERT期间自动生成，另一列需要触发器

.. literalinclude:: /_static/code/orm_fetch_server_defaults.py
   :language: python
   :linenos:


__mapper_args__ = {"eager_defaults": True}

如果映射类类似于上述模式，ORM为INSERT和UPDATE生成的SQL将在RETURNING子句中包含“created”和“updated”：

.. sourcecode:: sql

  INSERT INTO my_table (created) VALUES (now()) RETURNING my_table.id, my_table.created, my_table.updated

  UPDATE my_table SET updated=now() WHERE my_table.id = %(my_table_id)s RETURNING my_table.updated


.. _orm_dml_returning_objects:


使用INSERT、UPDATE和ON CONFLICT（即upsert）返回ORM对象
=======================================================

SQLAlchemy 2.0增强了触发几种ORM-enabled INSERT、UPDATE和upsert语句的能力。参见文档  :doc:`queryguide/dml`  进行文档化。有关upsert，请参见   :ref:` orm_queryguide_upsert` 。

使用带有RETURNING的PostgreSQL ON CONFLICT返回upserted ORM对象
------------------------------------------------------------

本节移到了   :ref:`orm_queryguide_upsert` 。


.. _session_partitioning:

分区策略（例如 Session 上的多个数据库后端）
==================================================

简单的垂直分区
-----------------

垂直分区使用  :paramref:`.Session.binds`  参数通过将不同的类、类层次结构或映射表配置在多个数据库中来划分它们。此参数接收一个字典，其中包含任意组合的ORM映射类、映射层次结构中的任意类（例如声明基类或mixin）、  :class:` _schema.Table`  对象和   :class:`_orm.Mapper`  对象作为键，然后通常引用   :class:` _engine.Engine`  或较少用   :class:`_engine.Connection`  对象作为目标。每当   :class:` .Session`  需要代表特定类型的映射类发出SQL以查找相应的数据库连接时，都会使用该字典。例如：

	engine1 = create_engine("postgresql+psycopg2://db1")
	engine2 = create_engine("postgresql+psycopg2://db2")

	Session = sessionmaker()

	# 将 User 操作绑定到 engine1，将 Account 操作绑定到 engine2。
	Session.configure(binds={User: engine1, Account: engine2})

	session = Session()

以上，针对任一类进行的 SQL 操作都将使用链接到该类的   :class:`_engine.Engine` 。此功能在读取和写入操作方面是综合性的。对映射到“engine1”（通过查看请求项列表中的第一个实体来确定）的实体进行的   :class:` _query.Query`  将使用“engine1”来运行查询。在每个类的基础上进行 flush 操作时，会单独使用 **两个** 引擎作为它刷新类型为“User”和“Account”的对象。

在较常见的情况下，通常有基础类或 mixin 类，可用于区分注定要发送到不同数据库连接的操作。  :paramref:`.Session.binds`   参数可以接受任意任意 Python 类作为键，如果发现它在特定映射类的 ` `__mro__``（Python 方法解析顺序）中，则将其用作键。假设两个声明基类表示两个不同的数据库连接：

	from sqlalchemy.orm import DeclarativeBase
	from sqlalchemy.orm import Session


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

	# 所有 User/Address 操作都将在 engine1 上执行，所有 Game 操作都将在 engine2 上执行。
	Session.configure(binds={BaseA: engine1, BaseB: engine2})


以上，从 ``BaseA`` 和 ``BaseB`` 派生的类将根据其来源父类（如果有的话）被路由到两个引擎之一。如果一个类从多个“绑定”超类派生，则将选择该目标类层次结构中最高的超类代表应使用的引擎。

.. seealso::

	 :paramref:`.Session.binds` 


为多元引擎会话协调事务
--------------------------

使用多个绑定引擎的一个警告是，在一个提交操作在一个后端成功提交之后，依然有可能在另一个后端中失败（译者注：即无法保证 ACID 特性中的 C 与 I）。这是一个不一致性问题，在关系型数据库中通过“两段事务”来解决，它向提交序列中添加了一个附加的“准备”步骤，以使多个数据库在实际完成事务之前就达成协议。

由于 DBAPI 的支持有限，SQLAlchemy 在跨多个后端使用两段事务方面的支持有限。通常，它已知可以在 PostgreSQL 后端上很好地使用，在 MySQL 后端上使用较小。但是，  :class:`.Session`  完全能够利用后端支持时的两阶段事务特性，方法是将  :paramref:` .Session.use_twophase`  标志设置为   :class:`.sessionmaker`  或   :class:` .Session`  中。请参见   :ref:`session_twophase`  进行示例。

.. _session_custom_partitioning:

自定义垂直分区
-------------------

通过覆盖  :meth:`.Session.get_bind`  方法可以构建更全面的基于规则的类级分区。以下我们将说明一种定制的   :class:` .Session` ，它提供以下规则：

1. Flush 操作以及批量“update”和“delete”操作均在名为“leader”的引擎上接收。

2. 所有子类为“MyOtherClass”的对象操作均发生在“other”引擎上。

3. 所有其他类的读操作在“follower1”或“follower2”数据库的随机选择上执行。例如：

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
	            return engines["leader"]
	        else:
	            return engines[random.choice(["follower1", "follower2"])]

上面的   :class:`.Session`  类使用 ` `class_`` 参数插入到   :class:`.sessionmaker`  中：

	Session = sessionmaker(class_=RoutingSession)

该方法与多个   :class:`_schema.MetaData`  对象结合使用，类似于使用声明性 ` `__abstract__`` 关键字的方法，如   :ref:`declarative_abstract`  中所述。

.. note:: 上面的示例说明了如何基于重写  :meth:`.Session.get_bind`  方法来将特定的 SQL 语句路由到所谓的“leader”或“follower”数据库上，根据语句是否期望写入数据来判断。但是，这似乎不是实际操作的最佳选择，因为它在读取和写入操作之间产生协调不一致。在实践中，最好根据正在进行的整体操作/事务构建   :class:` _orm.Session` ，作为“读取”或“写入”会话，以进行一致性处理。这样，将写入数据的操作也会在统一的事务范围内发出它们的读取查询。请参见   :ref:`session_transaction_isolation_enginewide`  中的示例，以获取使用自动提交连接设置一个 “只读” 操作的   :class:` _orm.sessionmaker`  和另一个会包含 :keyword:`DML` / COMMIT 操作的 “写入” 操作的配方。

.. seealso::

	`SQLAlchemy 中的 Django 风格的数据库路由器 <https://techspot.zzzeek.org/2012/01/11/django-style-database-routers-in-sqlalchemy/>`_  - 博客文章介绍了  :meth:`.Session.get_bind`  更全面的示例

水平分区
----------

水平分区将单个表（或一组表）的行跨多个数据库进行分区。SQLAlchemy   :class:`.Session`  包含这个概念的支持，但要想充分使用，需要使用   :class:` .Session`  和   :class:`_query.Query`  的子类。这些子类的基本版本可在   :ref:` horizontal_sharding_toplevel`  中找到。使用示例请参见：  :ref:`examples_sharding` 。

.. _bulk_operations:

批量操作
=========

.. legacy::

	SQLAlchemy 2.0 把 “批量插入”和“批量更新”能力集成到 2.0 风格的  :meth:`_orm.Session.execute`  方法中，直接使用   :class:` _dml.Insert`  和   :class:`_dml.Update`  构造体。请参见文档  :doc:` queryguide/dml`  进行文档化，包括   :ref:`orm_queryguide_legacy_bulk_insert` ，其中说明了从旧方法迁移到新方法的步骤。