.. _migration_20_toplevel:

======================================
SQLAlchemy 2.0 - 主要迁移指南
======================================

.. admonition:: 读者注意事项

    SQLAlchemy 2.0的转换文档分成**两个**文档 - 一个细节描述了从1.x到2.x的主要API转换系列，
    另一个则描述了与SQLAlchemy 1.4相比的新特性和行为：

    * :ref:`migration_20_toplevel` - 本文档，从1.x到2.x的API转换
    * :ref:`whatsnew_20_toplevel` - SQLAlchemy 2.0的新特性和行为

    它们都是适用于已经将1.4应用程序更新为遵循SQLAlchemy 2.0引擎和ORM约定的读者，为SQLAlchemy 2.0的新特性和功能提供概述。

.. admonition:: 关于本文档

    本文档描述了SQLAlchemy版本1.4与SQLAlchemy版本2.0之间的更改。

    SQLAlchemy 2.0为核心和ORM组件中的许多关键SQLAlchemy使用模式的一种结构性变革。本次更新的目标是在SQLAlchemy早期以来的若干项最基本的假设上进行轻微调整，并提供新的流畅的使用模型，通过这种方式，与Core和ORM组件之间的极简主义和一致性以及更强大的能力（以及更为标准化的能力）有望得到显著提高。将Python移动为Python 3only以及针对Python 3的渐进式类型系统的出现是这种转变的初始灵感，Python社区的变化以及不仅包括另一大批数据库程序员，而且还包括众多不同学科的数据科学家和学生。

    SQLAlchemy始于Python 2.3，当时没有上下文管理器，没有函数装饰器，Unicode只是一个第二类的功能，以及其他一些当今不知道的缺点。在SQLAlchemy 2.0中，最大的变化针对的是早期阶段SQLAlchemy开发中剩余的假设以及由于关键API特性的增量引入（例如：class:`.orm.query.Query` 和声明式）。它也希望标准化一些新的被证明非常有效的能力。

1.4->2.0迁移路径
---------------------------

被视为“SQLAlchemy 2.0”的最重要的体系结构特性和API更改实际上是作为完全可用的SQLAlchemy 1.4系列发布的，以提供从1.x到2.x系列的清洁升级路径，并作为功能本身的Beta平台。这些更改包括：

* :ref:`New ORM statement paradigm <change_5159>`
* :ref:`SQL caching throughout Core and ORM <change_4639>`
* :ref:`New Declarative features, ORM integration <change_5508>`
* :ref:`New Result object <change_result_14_core>`
* :ref:`select() / case() Accept Positional Expressions <change_5284>`
* :ref:`asyncio support for Core and ORM <change_3414>`

以上各项目录链接到SQLAlchemy 1.4中引入这些新范式的描述中。 :ref:`migration_14_toplevel` 。

对于SQLAlchemy 2.0，所有被标记为 :ref:`deprecated for 2.0 <deprecation_20_mode>` 的API特性和行为现在都已最终确定; 特别是，不再存在的主要API包括：

* :ref:`Bound MetaData and connectionless execution <migration_20_implicit_execution>`
* :ref:`Emulated autocommit on Connection <migration_20_autocommit>`
* :ref:`The Session.autocommit parameter / mode <migration_20_session_autocommit>`
* :ref:`List / keyword arguments to select() <migration_20_5284>`
* Python 2的支持

以上各项目录中的内容是最显著的完全向后不兼容的更改， 它们进入2.0的发行中被最终决定。要适应这些变化以及其他变化的应用程序迁移路径，首先定义为过渡路径与1.4系列的环境相结合，该系列中“将来”API可用，以提供“2.0”工作方式，然后移动到删除的应用程序API。上述标头引用了此迁移路径的完整步骤 :ref:`migration_20_overview` 。

.. _migration_20_overview:

1.x -> 2.x迁移概述
-----------------------------

SQLAlchemy 2.0转换在SQLAlchemy 1.4发行版中呈现出来，作为一系列步骤，允许使用渐进的迭代过程将任何规模或复杂度的应用程序迁移到SQLAlchemy 2.0。从Python 2到Python 3转换中学到的教训启发了一个系统，其可能性尽可能不需要任何“破坏性”更改，或必须普遍更改或不做任何更改。

作为检验2.0架构并允许完全迭代的转换环境的一种手段，整个2.0的新API和功能范围都存在并可用于1.4系列；这包括Core库的新功能领域，如SQL缓存系统，新的ORM声明性执行模式，用于ORM和Core的新型事务范式，统一经典映射和声明映射的新型ORM声明性系统，支持Python数据类以及Core和ORM的asyncio支持。

实现2.0迁移的步骤在以下各小节中进行了说明。总体而言，总体策略是一旦应用程序在1.4上运行，并开启了所有警告标志，并且不发出任何2.0-deprecation警告，那么它现在就**基本**与SQLAlchemy 2.0跨兼容了。 **请注意：可能会有附加的API和行为更改，当运行针对SQLAlchemy 2.0的代码时，可能会有不同的行为；在迁移的最后一步中始终针对实际的SQLAlchemy 2.0版本测试代码**。

第一个先决条件，步骤一 - 可工作的1.3应用程序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

对于典型的非平凡应用程序而言，使其在1.4上运行的第一步是确保其可以在SQLAlchemy 1.3上运行且没有deprecation警告。发布版1.4确实有一些与之前版本相关的警告，包括在1.3中发布的一些警告，特别是与:paramref:`_orm.relationship.viewonly`和:paramref:`_orm.relationship.sync_backref`标志相关的一些更改。

为了获得最佳结果，应用程序应该能够在最新的SQLAlchemy 1.3版本上运行，或者至少对至少使用Python 3.6的Python 3系列的SQLAlchemy无deprecation warnings。这些警告会发出:class:`_exc.SADeprecationWarning`类的警告。

第一个先决条件，第二步 - 可工作的1.4应用程序
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

一旦程序可在SQLAlchemy 1.3上使用，下一步是让其在SQLAlchemy 1.4上运行。在绝大多数情况下，应用程序应该可以在从SQLAlchemy 1.3到1.4的过程中没有问题地运行。 但是，在任何1.x和1.y版本之间的情况下，API和行为都会发生变化，或者在某些情况下更微妙地发生变化。 SQLAlchemy项目总是收到了大量回归报告的情况，这是在最初几个月中发生的现象。

1.x->1.y发行版通常在边缘周围进行了一些更改，这些更改基于预计几乎不会或根本不会使用的用例。对于1.4，被识别为位于此领域的更改如下：

* :ref:`change_5526` - 这影响对:class:`_engine.URL`对象进行操作的代码，并可能影响使用:class:`_engine.CreateEnginePlugin`扩展点的代码。这是一个不常见的情况，但可能会对特殊数据库配置逻辑使用的某些测试套件产生影响。通过对使用相对较新而又鲜为人知的:class:`_engine.CreateEnginePlugin`类的代码进行github搜索，找到了两个未受此更改影响的项目。

* :ref:`change_4617` - 此更改可能会影响在某种程度上依赖于:class:`_sql.Select`构造的大部分代码，其中会创建通常会导致混淆且不起作用的匿名子查询。除了SQLite外，大多数数据库都会拒绝这些子查询，因为通常需要名称，但仍有可能需要调整某些无意中依赖这些查询的查询的查询。

* :ref:`change_select_join` - 相关的，:class:`_sql.Select`类特别具有``.join()``和``.outerjoin()``方法，它们隐含地创建子查询，然后返回:class:`_sql.Join` 构造，这会导致大多数无用且产生大量混乱。决定采用极为有用的2.0风格连接构建方法前进，这里的这些方法现在与ORM :meth:`_orm.Query.join`方法的工作方式相同。

* :ref:`change_deferred_construction` - 与:class:`_orm.Query`或:class:`_sql.Select`的构造有关的一些错误消息可能不会在构造时间而是在编译/执行时发送。这可能会影响一些测试套件，这些套件针对失败模式进行测试。

更改的完整概述请参见 :doc:`/changelog/migration_14` 文档。

迁移到2.0步骤一 - 仅使用Python 3（Python 3.7是2.0兼容性的最低要求）
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 2.0最初受到Python 2的EOL是在2020年的事实的启发。相对于其他主要项目而言，SQLAlchemy需要比较长的时间才能放弃对Python 2.7的支持。但是，为了使用SQLAlchemy 2.0，应用程序需要在至少**Python 3.7**上运行。 SQLALchemy 1.4支持Python 3系列中的Python 3.6或更新版本；在1.4系列的整个过程中，应用程序可以继续运行在Python 2.7或至少Python 3.6上。但以Python 3.7开始的SQLAlchemy 2.0。

.. _migration_20_deprecations_mode:

迁移到2.0步骤二 - 打开RemovedIn20Warnings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SQLAlchemy 1.4具有一种条件性退化警告系统，该系统受启发于Python“-3”标记，该标记将在运行应用程序时指示遗留模式。对于SQLAlchemy 1.4，只有当环境变量“SQLALCHEMY_WARN_20”设置为“true”或“1”时，才会发出 :class:`_exc.RemovedIn20Warning`退化类。

给定下面的示例程序：

    from sqlalchemy import column
    from sqlalchemy import create_engine
    from sqlalchemy import select
    from sqlalchemy import table


    engine = create_engine("sqlite://")

    engine.execute("CREATE TABLE foo (id integer)")
    engine.execute("INSERT INTO foo (id) VALUES (1)")


    foo = table("foo", column("id"))
    result = engine.execute(select([foo.c.id]))

    print(result.fetchall())

上面的程序使用许多许多用户已经将其视为“遗留”的模式，即包括:meth:`_engine.Engine.execute`方法的使用（这是“无连接执行”API的一部分）。当我们在1.4上运行上面的程序时，它返回单个行：

.. sourcecode:: text

  $ python test3.py
  [(1,)]

为了启用“2.0 deprecations mode”，我们启用“SQLALCHEMY_WARN_20=1”变量，另外确保选择不会抑制任何警告的警告过滤器： 

.. sourcecode:: text

    SQLALCHEMY_WARN_20=1 python -W always::DeprecationWarning test3.py

由于报告的警告位置不总是正确的位置，因此可能很难找到引发错误的代码。可以通过指定本文档中描述的 :class:`_exc.RemovedIn20Warning` 的 ``error`` 警告过滤器将警告转换为异常来实现。

使用打开警告的方式，我们的程序会发出很多更严重的警告：

.. sourcecode:: text

  $ SQLALCHEMY_WARN_20=1 python2 -W always::DeprecationWarning test3.py
  test3.py:9: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    engine.execute("CREATE TABLE foo (id integer)")
  /home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    return connection.execute(statement, *multiparams, **params)
  /home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    self._commit_impl(autocommit=True)
  test3.py:10: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    engine.execute("INSERT INTO foo (id) VALUES (1)")
  /home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:2856: RemovedIn20Warning: Passing a string to Connection.execute() is deprecated and will be removed in version 2.0.  Use the text() construct, or the Connection.exec_driver_sql() method to invoke a driver-level SQL string. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    return connection.execute(statement, *multiparams, **params)
  /home/classic/dev/sqlalchemy/lib/sqlalchemy/engine/base.py:1639: RemovedIn20Warning: The current statement is being autocommitted using implicit autocommit.Implicit autocommit will be removed in SQLAlchemy 2.0.   Use the .begin() method of Engine or Connection in order to use an explicit transaction for DML and DDL statements. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    self._commit_impl(autocommit=True)
  /home/classic/dev/sqlalchemy/lib/sqlalchemy/sql/selectable.py:4271: RemovedIn20Warning: The legacy calling style of select() is deprecated and will be removed in SQLAlchemy 2.0.  Please use the new calling style described at select(). (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    return cls.create_legacy_select(*args, **kw)
  test3.py:14: RemovedIn20Warning: The Engine.execute() function/method is considered legacy as of the 1.x series of SQLAlchemy and will be removed in 2.0. All statement execution in SQLAlchemy 2.0 is performed by the Connection.execute() method of Connection, or in the ORM by the Session.execute() method of Session. (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9) (Background on SQLAlchemy 2.0 at: https://sqlalche.me/e/b8d9)
    result = engine.execute(select([foo.c.id]))
  [(1,)]

有了上面的指导，我们可以将程序迁移到使用2.0风格，并且作为奖励，我们的程序更加清晰：

    from sqlalchemy import column
    from sqlalchemy import create_engine
    from sqlalchemy import select
    from sqlalchemy import table
    from sqlalchemy import text


    engine = create_engine("sqlite://")

    # 不要依赖自动提交的DML和DDL
    with engine.begin() as connection:
        # 使用connection.execute()，而不是engine.execute()
        # 使用text()构造执行文本SQL
        connection.execute(text("CREATE TABLE foo (id integer)"))
        connection.execute(text("INSERT INTO foo (id) VALUES (1)"))


    foo = table("foo", column("id"))

    with engine.connect() as connection:
        # 使用connection.execute()，而不是engine.execute()
        # 现在select()可以接受位置表达式的列/表达式
        result = connection.execute(select(foo.c.id))

    print(result.fetchall())

“2.0退化模式”的目标是，当一个程序在打开“2.0退化模式”的情况下运行时，不发出 :class:`_exc.RemovedIn20Warning` 警告，程序现在已经准备好在SQLAlchemy 2.0中运行。

迁移到2.0步骤三 - 解决所有RemovedIn20Warnings
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

可以向代码中迭代开发解决这些警告。 在SQLAlchemy项目本身中，采用以下方法：

1.在测试套件中启用“SQLALCHEMY_WARN_20=1”环境变量，对于SQLAlchemy而言，这在tox.ini文件中。

2.在设置测试套件时，建立一系列警告过滤器，以选择特定子集的警告来引发异常或忽略（或記入日志）。每次处理一个警告子组。

3.当解决应用程序中的每个子类别的警告时，可以将新的警告加入到要解决的“错误”列表中，这些警告被“始终”过滤器捕获。

4.一旦不再有警告发出，就可以删除过滤器。

迁移到2.0步骤四 - 在Engine上使用“future”标志
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`_engine.Engine` 对象具有更新的事务级API，在SQLAlchemy 2.0中，将不会自动提交数据库API事务，这与SQLAlchemy 1.x中的会自动提交::

    conn = engine.connect()


自动提交

在 2.0 中，执行数据库插入、更新或删除操作时，不会自动提交：

    # 不会自动提交，只是插入了一条记录到数据库中
    conn.execute(some_table.insert().values(foo="bar"))

同样，这个也不会自动提交：

    conn = engine.connect()

    # 不会自动提交，但是只插入了一条记录到数据库中
    conn.execute(text("INSERT INTO table (foo) VALUES ('bar')"))

这里提供一种常见的解决方法，当需要提交的定制化 DML 语句构成时，需要使用“autocommit”
选项，这个选项将被删除：

    conn = engine.connect()

    # 在 2.0 中不会自动提交
    conn.execute(text("EXEC my_procedural_thing()").execution_options(autocommit=True))

2.0迁移

此操作既适用于 1.x 风格，也适用于 2.0 风格执行，最好使用 :meth:`_engine.Connection.begin`
方法或 :meth:`_engine.Engine.begin` 上下文管理器。

    with engine.begin() as conn:
        conn.execute(some_table.insert().values(foo="bar"))
        conn.execute(some_other_table.insert().values(bat="hoho"))

    with engine.connect() as conn:
        with conn.begin():
            conn.execute(some_table.insert().values(foo="bar"))
            conn.execute(some_other_table.insert().values(bat="hoho"))

    with engine.begin() as conn:
        conn.execute(text("EXEC my_procedural_thing()"))

与 :term:`2.0 风格`一起使用 :paramref:`_sa.create_engine.future` 标记时，"commit as you go" 风格也可以使用，因为
:class:`_engine.Connection` 包含了自动开始处理的 **autobegin** 行为，当一条语句在未明确调用:meth:`_engine.Connection.begin`
的情况下首次被调用时，这个特性就会生效：

    with engine.connect() as conn:
        conn.execute(some_table.insert().values(foo="bar"))
        conn.execute(some_other_table.insert().values(bat="hoho"))

        conn.commit()

启用 :ref:`2.0 deprecations mode <migration_20_deprecations_mode>` 后，在使用"autocommit"特性时将发出警告，指出需要注意哪些地方需要注明显式事务。

讨论

SQLAlchemy 最初的几个版本与 Python DBAPI(:pep:`249`) 的精神不符，因为它试图隐藏 :pep:`249` 强调的“隐式开始”和“显式提交”事务的问题。
十五年后，我们发现这是一个错误，因为 SQLAlchemy 的许多试图“隐藏”事务存在的模式构成了一个更复杂的 API，工作方式不一致，对于那些新接触关系数据库和 ACID 事务的用户来说非常混乱。 SQLAlchemy 2.0 将消除所有自动提交事务的尝试，使用模式将始终要求用户以某种方式划分事务的“开始”和“结束”，就像 Python 中读写文件一样，有一个“开始”和“结束”的过程。

对于纯文本语句的自动提交，实际上有一个正则表达式来分析每个语句，以检测自动提交！并不奇怪，这个正则表达式不断失败，无法适应各种暗示“写入”到数据库中的语句和存储过程，导致我们一直困惑于某些语句在数据库中产生结果，而其他语句则没有结果。通过防止用户了解事务概念，我们得到了很多与此相关的错误报告，因为用户不理解，无论是否某个层级正在自动提交，数据库始终使用事务。

SQLAlchemy 2.0 将要求在每个级别的所有数据库操作中显式地指定如何使用事务。对于大多数 Core 使用案例，推荐使用的模式是：

    with engine.begin() as conn:
        conn.execute(some_table.insert().values(foo="bar"))

对于“commit as you go 或者回滚”用法，它类似于如何使用 ORM 级 :class:`_orm.Session`，这个 "future" 版本的 :class:`_engine.Connection`，这是从使用的 :class:`_engine.Engine` 返回的一个，engine 是使用 :paramref:`_sa.create_engine.future` 标志创建的。这个版本包括新的 :meth:`_engine.Connection.commit` 和 :meth:`_engine.Connection.rollback` 方法，它们基于当第一次执行语句时自动开始的事务进行操作：

    # 1.4 / 2.0 代码

    from sqlalchemy import create_engine

    engine = create_engine(..., future=True)

    with engine.connect() as conn:
        conn.execute(some_table.insert().values(foo="bar"))
        conn.commit()

        conn.execute(text("some other SQL"))
        conn.rollback()

需要注意的是，``engine.connect()`` 方法将返回一个 :class:`_engine.Connection`，它具有 **autobegin** 特性，这意味着
``begin()`` 事件在第一次使用 execute 方法时被调用（请注意，Python DBAPI 中没有实际的 "BEGIN"）。"autobegin" 是 SQLAlchemy 1.4
的新模式，它被 :class:`_engine.Connection` 以及 ORM :class:`_orm.Session` 对象同时支持；
autobegin 允许在对象第一次被获取时显示调用 :meth:`_engine.Connection.begin` 方法，以适应需要划分事务的方案，但如果不调用该方法，则会在对象的第一次操作时隐式执行。

去掉 "autocommit" 与去掉 "connectionless" 执行、"bound metadata"

**概述**

将 :class:`_engine.Engine` 与 :class:`_schema.MetaData` 对象关联起来，使“connectionless”执行模式得到了解决，这种模式可以使用一系列所谓的“connectionless” 执行模式，现在被取消。使用 :class:`_orm.Session.execute` 方法执行语句时， :class:`_engine.Engine` 来处理连接资源， :class:`_schema.MetaData.create_all` 操作始终使用 :class:`_engine.Engine` 获取连接。现在使用 :class:`_engine.Connection`对象执行语句后，只需创建 :class:`_engine.Connection`，以开启事务块。

**迁移至 2.0**

只能使用 :class:`_engine.Engine` 或 :class:`_engine.Connection` 对象来执行数据库表的创建与映射，执行语句时只有 :class:`_engine.Connection` 对象有 :meth:`_engine.Connection.execute` 方法。与编辑 :class:`_schema.MetaData` 或签入/签出交互的方式相比，代码用法会更加明确。

    from sqlalchemy import MetaData

    metadata_obj = MetaData()

    # engine 级 operation:

    # 创建表
    metadata_obj.create_all(engine)

    # 反射所有表结构
    metadata_obj.reflect(engine)

    # 反射单个表结构
    t = Table("t", metadata_obj, autoload_with=engine)


    # connection 级 operation:

    with engine.connect() as connection:
        # 创建表，需要显式的 begin() 和/或 commit()
        with connection.begin():
            metadata_obj.create_all(connection)

        # 反射所有表结构
        metadata_obj.reflect(connection)

        # 反射单个表结构
        t = Table("t", metadata_obj, autoload_with=connection)

        # 执行 SQL 语句
        result = connection.execute(t.select())

**讨论**

- 此更改会移除 :class:`_orm.Session.execute` 方法的 "connectionless" 执行模式。
- 执行方法更改为 "generative" 风格。
- "bound metadata" 现在已无用武之地。
- 数据库表和映射关系现在只能使用 :class:`_engine.Connection` 创建，不能使用 :class:`_engine.Engine`。

废弃 :meth:`_engine.Connection.execute`、执行选项更加突出

**概述**

在 SQLAlchemy 2.0 中，传递给 :meth:`_engine.Connection` execute 方法的参数模式得到极大的简化，除了 table 外的参数构造不再存在：

    # 不再支持直接插入
    stmt = insert(table, values={"x": 10, "y": 15}, inline=True)

    # 不再支持直接插入
    stmt = insert(table, values={"x": 10, "y": 15}, returning=[table.c.x])

    # 不再支持 table.delete(table.c.x > 15)
    stmt = table.delete(table.c.x > 15)

    # 不再支持
    stmt = table.update(table.c.x < 15, preserve_parameter_order=True).values(
        [(table.c.y, 20), (table.c.x, table.c.y + 10)]
    )

**迁移至 2.0**

下面的示例说明了这里面的例子应该如何移植：

    # 使用 generative 方法
    stmt = insert(table).values(x=10, y=15).inline()

    # 使用 generative 方法，字典仍然 OK
    stmt = insert(table).values({"x": 10, "y": 15}).returning(table.c.x)

    # 使用 generative 方法
    stmt = table.delete().where(table.c.x > 15)

    # 使用 generative 方法，ordered_values() 代替 preserve_parameter_order
    stmt = (
        table.update()
        .where(
            table.c.x < 15,
        )
        .ordered_values((table.c.y, 20), (table.c.x, table.c.y + 10))
    )

**讨论**

与 :func:`_sql.select` 一样，SQLAlchemy 已经发展出了一种行为惯例，即构造元素列表的传递方式，建构元素应该使用位置传递方式。最终，SQLAlchemy 2.0将解决 :func:`_sql.select` 构造的问题，因为历史遗留的调用方式中的 "WHERE" 子句是通过位置传递的。

**变动包括**

- 删除了插入、更新或删除 DML 的非 table 参数构造函数。
- :meth:`_engine.Connection.execute` 不再允许使用 *args 和 **kwargs。这些参数已经被移除，为选项传递留下了容量。
- :meth:`_orm.Session.execute` 现在已暴露为一种 generative 写作模式，就像大多数 ORM 查询一样。只允许使用字典来传递构造参数。

结果行像命名元组一样

**概述**

一个新的 :class:`_engine.Row` 类被与支持 “future” 模式的 :class:`_engine.Engine`，:class:`_engine.Connection` 和 :class:`_query.Query` 关联的结果对象一般返回，:class:`_engine.Row` 的行为像命名元组一样，以“future”模式使用 :meth:`_engine.Engine` 或 :class:`_engine.Connection`，但在 1.x版本中，默认的是返回 SQL 中变量名称为 key，这个变化是不兼容的。

**迁移至 2.0**

如果需要从结果行中获取某个键,应该使用``row.keys()`。但这是一个异常情况，因为结果行通常是由已知的列组成的。

**讨论**

SQLAlchemy 1.4 引入了全新的 :class:`_engine.Row` 实现，它是由
:class:`_query.Query` 在选择行时返回的，这个类行为类似于命名元组。然而，它还提供先前的“映射”行为，方法是提供一个特别的属性 row._mapping，这个属性可以产生一个 Python 映射，以便可以使用这样的 keyed 访问方式 row["some_column"]。

:class:`_engine.Row` 还支持通过实体或属性进行访问，例如：``row.some_column``。
例如，在数据库中的应用架构中，已知存在一组列。仅在非常罕见的情况下，结果行被用于测试是否存在某个键。

.. seealso::

    :ref:`change_4710_core`
    更多了解请到官方文档去查看
    ../../orm/extensions/declarative/bases.rst#sqlalchemy.ext.declarative.api.Base

declarative_base() 成为第一类 API

**概述**

``sqlalchemy.ext.declarative``
功能模块的大部分，除了一些例外，被移动到 ``sqlalchemy.orm`` 导入混合类型等用法。

**迁移到 2.0**

更改导入：

    from sqlalchemy.ext import declarative_base, declared_attr

为：

    from sqlalchemy.orm import declarative_base, declared_attr

**讨论**

在多年的流行之后，``sqlalchemy.ext.declarative`` 模块现在已整合到 ``sqlalchemy.orm`` 命名空间中，但除了声明扩展类以外其他的东西都不影响。更改的详细信息在 1.4 迁移指南上有说明。
声明式、经典映射、数据类、attrs等。

:ref:`change_5508`

声明式的核心元素现在将原来的 standalone 函数"mapper()"移至后台，由更高级别的 API 调用。“mapper()”方法被移植到了 _orm.registry.map_imperatively，该方法是由 _orm.registry 对象生成的。

2.0的迁移
与经典映射一起工作的代码应该从以下格式进行调整：

    from sqlalchemy.orm import mapper


    mapper(SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)})
    
工作方式调整为以中央 _orm.registry 对象为核心：

    from sqlalchemy.orm import registry

    mapper_reg = registry()

    mapper_reg.map_imperatively(
        SomeClass, some_table, properties={"related": relationship(SomeRelatedClass)}
    )

上述 _orm.registry 对象也是声明式映射的来源，现在经典映射也可以使用该注册表，包括对 _orm.relationship 进行基于字符串的配置。

    from sqlalchemy.orm import registry

    mapper_reg = registry()

    Base = mapper_reg.generate_base()


    class SomeRelatedClass(Base):
        __tablename__ = "related"

        # ...
    
    
    mapper_reg.map_imperatively(
        SomeClass,
        some_table,
        properties={
            "related": relationship(
                "SomeRelatedClass",
                primaryjoin="SomeRelatedClass.related_id == SomeClass.id",
            )
        },
    )

该经典映射设计的主要理念在于将 _schema.Table 的设置与类别分开，而声明式一直以来都可以使用这种风格。

此外，为了消除基类要求，增加了一种一类别装饰器形式。作为其他一些增强功能，支持 Python 数据类，这种数据类可以使用声明式装饰器和经典映射形式。

有关所有新的声明式和经典映射、数据类、attrs等的统一文档，请参见 : ref: `orm_mapping_classes_toplevel`。

.. _migration_20_query_usage:

ORM使用2.0迁移
---------------------------------------------

SQLAlchemy 2.0中最显著的可见变化是使用 :meth:`_orm.Session.execute` 结合 :func:`_sql.select` 运行 ORM 查询，而不是使用 :meth:`_orm.Session.query`。正如其他讨论所提到的，实际上没有计划彻底删除 :meth:`_orm.Session.query` API 本身，因为现在它是以新的API内部实现为基础，因此它会保留为遗留API，可以自由地使用两个API。

下表提供了一般呼叫形式的介绍以及链入对每种技术呈现的文档的链接。嵌入性节部分中的个别迁移说明可能包含这里未总结的其他说明。

.. format: off

.. container:: sliding-table

  .. list-table:: ** ORM查询模式概述**
    :header-rows: 1

    * - :term:`1.x式` 格式
      - :term:`2.0式` 格式
      - 参见

    * - ::

          session.query(User).get(42)

      - ::

          session.get(User, 42)

      - :ref:`migration_20_get_to_session`

    * - ::

          session.query(User).all()

      - ::

          session.execute(
            select(User)
          ).scalars().all()

          # 或者

          session.scalars(
            select(User)
          ).all()

      - :ref:`migration_20_unify_select`

        :meth:`_orm.Session.scalars`
        :meth:`_engine.Result.scalars`

    * - ::

          session.query(User).\
            filter_by(name="some user").\
            one()

      - ::

          session.execute(
            select(User).
            filter_by(name="some user")
          ).scalar_one()

      - :ref:`migration_20_unify_select`

        :meth:`_engine.Result.scalar_one`

    * - ::

          session.query(User).\
            filter_by(name="some user").\
            first()

      - ::

          session.scalars(
            select(User).
            filter_by(name="some user").
            limit(1)
          ).first()

      - :ref:`migration_20_unify_select`

        :meth:`_engine.Result.first`

    * - ::

            session.query(User).options(
              joinedload(User.addresses)
            ).all()

      - ::

            session.scalars(
              select(User).
              options(
                joinedload(User.addresses)
              )
            ).unique().all()

      - :ref:`joinedload_not_uniqued`

    * - ::

          session.query(User).\
            join(Address).\
            filter(
              Address.email == "e@sa.us"
            ).\
            all()

      - ::

          session.execute(
            select(User).
            join(Address).
            where(
              Address.email == "e@sa.us"
            )
          ).scalars().all()

      - :ref:`migration_20_unify_select`

        :ref:`orm_queryguide_joins`

    * - ::

          session.query(User).\
            from_statement(
              text("select * from users")
            ).\
            all()

      - ::

          session.scalars(
            select(User).
            from_statement(
              text("select * from users")
            )
          ).all()

      - :ref:`orm_queryguide_selecting_text`

    * - ::

          session.query(User).\
            join(User.addresses).\
            options(
              contains_eager(User.addresses)
            ).\
            populate_existing().all()

      - ::

          session.execute(
            select(User)
            .join(User.addresses)
            .options(
              contains_eager(User.addresses)
            )
            .execution_options(
                populate_existing=True
            )
          ).scalars().all()

      -

          :ref:`orm_queryguide_execution_options`

          :ref:`orm_queryguide_populate_existing`

    *
      - ::

          session.query(User).\
            filter(User.name == "foo").\
            update(
              {"fullname": "Foo Bar"},
              synchronize_session="evaluate"
            )

      - ::

          session.execute(
            update(User)
            .where(User.name == "foo")
            .values(fullname="Foo Bar")
            .execution_options(
              synchronize_session="evaluate"
            )
          )

      - :ref:`orm_expression_update_delete`

    *
      - ::

          session.query(User).count()

      - ::

          session.scalar(
            select(func.count()).
            select_from(User)
          )

          # 或者

          session.scalar(
            select(func.count(User.id))
          )

      - :meth:`_orm.Session.scalar`

.. format: on

.. _migration_20_unify_select:

ORM查询与核心选择器统一
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**概要**

:class:`_query.Query` 对象（以及 :class:`_baked.BakedQuery` 和 :class:`_horizontal.ShardedQuery` 扩展）现在是遗留对象，被直接使用 :func:`_sql.select` 构造的结合 :meth:`_orm.Session.execute` 方法取代。从 :class:`_orm.Query` 中返回的结果以 :class:`_engine.Result` 对象形式统一进行返回，这些结果以对象列表、元组或标量 ORM 对象的形式返回，这些对象具有与 Core 执行相同的接口。

应使用映射属性来区分ORM对象和关系字符串中的多个属性，例如，不在支持:meth:`_query.Query.join` 和 :func:`_orm.joinedload` 上使用属性名称构成的字符串例子将被删除。

永远不要将对象从一个位置传递到另一个位置。需要从一个位置到另一个位置的是对象的唯一标识符，例如它的主键值等。

**迁移到2.0**

 :class:`_orm.Query` 的大部分应用现在可以使用 Core 的 :func:`_sql.select` 对象来实现，使用 :meth:`_orm.Session.execute` 运行 SQL 语句。例如，在 :term:`1.x式` 中，跨表查询可能像下面这样：

    session.query(User).join("addresses")

在 :term:`2.0式` 编写合法代码时，使用以下方式：

    select(User).join(User.addresses)

查询可以像下面这样执行：

    session.execute(select(User).join(User.addresses))

因为结果都是 :class:`_engine.Result` 对象，因此可以使用和 :meth:` _orm.Session.query` 相同的方法执行查询，但是使用执行器  :meth: ` _orm.Session.execute` 集成 :meth:`_sql.select`， 它具有不同的方法，例如 scalars(), unique() 等。有关 :func:`_sql.select` 的完整说明，请参见 :doc:`/orm/queryguide`。

以下是迁移 :func:`_sql.select` 的示例：
　　

    # SQLAlchemy 1.4 / 2.0 兼容
    stmt = select(User).join(User.addresses)
    result = session.execute(stmt)

    # 或
    session.execute(select(User).join(User.addresses)).scalars().all()

    # 使用唯一性 modifier
    session.execute(select(User).options(joinedload(User.addresses))).unique().all()
 

**讨论**

:class:`_query.Query` 和 :func:`_sql.select` 的重叠是 SQLAlchemy 最大的不协调的地方之一。这个问题产生的原因是由于逐步增加导致两个关键 API 发生分歧。

在 SQLAlchemy 的前几个版本中， :class:`_orm.Query` 对象甚至并不存在。最初的想法是 :class:`_orm.Mapper` 构建本身可以选择行，而不是类需要使用 :class:`_schema.Table`，创建核心风格的条件。 :class:`_query.Query` 几个月/几年内部产生了一些被接受的新功能，而成为一个新的“可构建性”的查询对象最初称为``SelectResults``。类似 "where()" 方法的概念，在 SQLAlchemy 中以前是不存在的， :func:`_sql.select` 仅使用一次-at-once 构造方式，在 :ref:`migration_20_5284` 中现在已经被弃用。

随着新方法的出现， :class:`_orm.Query` 对象向 :class:`_sql.select` 对象添加了很多新的方法，包括选择单个列、同时选择多个实体，从 :class:`_orm.Query` 对象构建子查询等等，这个目标就是 :class:`_orm.Query` 应该拥有 :class:`_sql.select` 的全部功能，可以构成完整的SELECT语句，无需使用 :func:`_sql.select` 外显。同时， :func:`_sql.select` 已经发展出了"generative"方法，例如 :meth:`_sql.Select.where` 和 :meth:`_sql.Select.order_by`。

现代 SQLAlchemy 已经实现了这个目标，两个对象现在完全重叠。统一的主要挑战在于 :func:`_sql.select` 对象需要始终 **完全脱离 ORM**。为了实现这一点，大部分 :class:`_orm.Query` 的逻辑已经从 SQL 编译阶段移动到了 SQL 编译阶段，其中 ORM 特定的编译器插件接收 :class:`_sql.Select` 构造，并解释它的内容，以ORM样式的查询表示，然后转移到内核级别编译器，以创建 SQL 字符串。随着新的SQL编译缓存系统的出现，大部分的 ORM 逻辑也被缓存。

.. 参见：

  :ref:`change_5159`

.. _migration_20_get_to_session:

ORM 查询 - get() 方法移至 Session
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**概要**

:meth:`_query.Query.get` 方法现在为遗留目的而保留，但主要接口现在是 :meth:`_orm.Session.get` 方法：

    #老用法
    user_obj = session.query(User).get(5)

**迁移到 2.0**

在 1.4 / 2.0 中， :class:`_orm.Session` 对象添加了新的 :meth:`_orm.Session.get` 方法：

    # 1.4 / 2.0 版本交叉兼容使用
    user_obj = session.get(User, 5)

**讨论**

:meth:`_query.Query.get` 方法定义了与 :class:`_orm.Session` 的特殊交互方式，而且可能甚至不需要发出查询，更适合它是 :class:`_orm.Session` 的一部分，类似于其他"identity"方法，例如 :class:`_orm.Session.refresh` 和 :class:`_orm.Session.merge`。
 
.. _migration_20_orm_query_join_strings:

ORM 查询 - Joining / loading on relationships 使用属性，而不是字符串
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**概要**

不再支持将字符串属性或实际类别属性混合使用，这涉及到诸如 :meth:`_query.Query.join` 这样的模式以及选项，例如 :func:`_orm.joinedload`。

    #删除字符串使用
    q = session.query(User).join("addresses")

    #删除字符串使用
    q = session.query(User).options(joinedload("addresses"))

    #删除字符串使用
    q = session.query(Address).filter(with_parent(u1, "addresses"))

**迁移到 2.0**

现代 SQLAlchemy 1.x 版本支持推荐的技术，即使用映射属性：

    #与所有现代SQLAlchemy版本兼容的代码
    q = session.query(User).join(User.addresses)

    q = session.query(User).options(joinedload(User.addresses))

    q = session.query(Address).filter(with_parent(u1, User.addresses))

相同的技术也适用于 :term:`2.0式` 的用法：

    from sqlalchemy.orm import joinedload

    session.query(User).join(User.addresses)

    session.query(Address).filter(with_parent(u1, User.addresses))

因此，所述 ORM 查询统一与 Core 的:模块:`_sql.select`，并允许 ORM 属性映射的属性同时充当列名和列对象。区分 ORM 对象和关系字符串中的多个属性所需的技术变成了使用属性本身，这一要求已经从该库中删除了在 1.xx 使用字符串属性的做法。

...参考文档

.. 参见::

  :ref:`change_5284`
  
.. _migration_20_query_join_options:

ORM 查询 - join(...，aliased=True)，from_joinpoint已去除
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**概要**

:class:`_query.Query.join` 上的 ``aliased=True`` 选项已删除，同时也删除了 ``from_joinpoint`` 标志：

    #不再支持
    q = (
        session.query(Node)
        .join("children", aliased=True)
        .filter(Node.name == "some sub child")
        .join("children", from_joinpoint=True, aliased=True)
        .filter(Node.name == "some sub sub child")
    )

**迁移到 2.0**

现在需要使用显式别名：

    n1 = aliased(Node)
    n2 = aliased(Node)

    q = (
        select(Node)
        .join(Node.children.of_type(n1))
        .where(n1.name == "some sub child")
        .join(n1.children.of_type(n2))
        .where(n2.name == "some sub child")
    )

**讨论**

:meth:`_query.Query.join` 上的 ``aliased=True`` 选项是极少被使用的一个选项, 基于代码搜索， 可以发现实际的使用率非常非常低。由于 ``aliased=True`` 如果需要变更过滤器条件，内部的复杂度非常高，将在 2.0 中消失。

大多数用户并不熟悉此标志，但它允许沿连接自动为元素添加别名，然后将筛选条件的自动别名扩展为实际结果。早期的用例是协助长串联自我参照连接，但是，筛选条件的自动适应非常复杂，几乎从未在现实世界的应用程序中使用。另外，该模式还存在问题，例如如果需要在链接链中的每个链接处添加过滤器条件，则该模式将必须使用 ``from_joinpoint`` 标志， 。 SQLAlchemy 的开发人员绝对没有在真实世界的应用程序中找到过此参数的任何用法。

``aliased=True`` 和 ``from_joinpoint`` 参数是在 :class:`_query.Query` 对象还没有可用于关注关系属性的功能（如 :meth:`.PropComparator.of_type`）以及 :func:`.aliased` 结构本身不存在早期开发期间开发的。在 SQLAlchemy 1.4 中，:func:`.aliased` 结构已经被增强，可以针对任何任意可选择项发出 ORM 查询，也可以多次对相同的子查询使用它，且纽结较多，可以被用于 :term:`1.x式` 的 :class:`_orm.Query`，在上述示例中，由于最后一个查询需要从多个表查询，因此需要创建两个单独的 :func:`_orm.aliased` 结构， :term:`2.0式` 的情况下也可以使用相同的形式。
...
.. _migration_20_query_distinct:

使用DISTINCT选项选择其他列，但仅选择实体
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

当前 :class:`_query.Query` 得到 distinct 使用时，在 ORDER BY 中自动添加列。以下查询将从所有 User 的列中选择，还选择“address.email_address”，但仅返回 User 对象:

    # 1.xx 代码

    result = (
        session.query(User)
        .join(User.addresses)
        .distinct()
        .order_by(Address.email_address)
        .all()
    )

在 2.0 中，"email_address" 列将不会自动添加到列子句中，因此上面的查询将失败，因为当使用 DISTINCT 时，关系数据库不允许对 "address.email_address" 进行排序，如果它没有在列子句中。

**迁移到 2.0**

在 2.0 中，必须明确添加该列。为了只返回主要实体对象而不返回附加列，请使用 :meth:`_result.Result.columns` 方法：

    #1.4 /2.0代码

    stmt = (
        select(User, Address.email_address)
        .join(User.addresses)
        .distinct()
        .order_by(Address.email_address)
    )

    result = session.execute(stmt).columns(User).all()

**讨论**

这种情况是 :class:`_query.Query` 的有限灵活性导致必须添加隐式的“神奇”行为;“email_address”列隐式添加到列子句中，然后其他的过滤条件逻辑从实际结果中省略该列。

新方法简化了交互并使正在进行的事情明确化，同时仍然可以实现原始用例而不会出现任何不便。

.. 参见：

  :ref:`change_5284`
  

.. _migration_20_query_from_self:

选择子查询自身作为查询，例如"from_self()"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

现在已删除 :meth:`_orm.Query.from_self` 方法：

    # from_self 被删除
    q = (
        session.query(User, Address.email_address)
        .join(User.addresses)
        .from_self(User)
        .order_by(Address.email_address)
    )

**迁移到 2.0**

现在使用 :func:`._orm.aliased` 结构可以用于在查询任意可选择项的实体时发出 ORM 查询。对它进行了增强，使其可以充分满足多次使用相同子查询以进行不同实体的查询的需要。这可以在 :term:`1.x式` 中使用 :class:`_orm.Query`:

   

   from sqlalchemy.orm import aliased

   subq = session.query(User, Address.email_address).join(User.addresses).subquery()

   ua = aliased(User, subq)

   aa = aliased(Address, subq)

   q = session.query(ua, aa).order_by(aa.email_address)

可在 :term:`2.0式` 中使用：

   

   from sqlalchemy.orm import aliased

   subq = select(User, Address.email_address).join(User.addresses).subquery()

   ua = aliased(User, subq)

   aa = aliased(Address, subq)

   stmt = select(ua, aa).order_by(aa.email_address)

   result = session.execute(stmt)

**讨论**

:meth:`_query.Query.from_self` 是一个非常复杂的方法，很少使用。该函数的目的是将 :class:`_query.Query` 转换为子查询，然后返回一个新的 :class:`_query.Query` 对象，该对象从该子查询中进行 SELECT。
因为 :meth:`_query.Query.from_self` 包含了大量的内置隐式翻译函数，则进入所产生的 SQL，尽管它允许使用较为简洁的方式来执行某种类型的查询，但此方法真正的现实使用也是不频繁的，因为它并不是一个易于理解的方法。

新方法使用 :func:`_orm.aliased` 结构，所以 ORM 的调用不再需要猜测哪些实体和列应该被适当修改，以何种方式进行适应; 在上面的示例中， "ua"和 "aa"对象均为 :class:`_orm.AliasedClass` 实例，它们为内部提供了一个明确的标记，以标明子查询应该被称为以及对于此查询的给定组件正在考虑哪个实体列或关系。


SQLAlchemy 1.4 还具有改进的标记样式，不再需要在标签中包含表名才能区分不同表中的相同列名的使用。在上述示例中，即使我们的 "User"和 "Address" 实体具有重叠的列名，我们也可以在不必指定任何特定标签的情况下选择两个实体：

    #1.4 /2.0代码

    subq = select(User, Address).join(User.addresses).subquery()

    ua = aliased(User, subq)
    aa = aliased(Address, subq)

    stmt = select(ua, aa).order_by(aa.email_address)
    result = session.execute(stmt)

上述查询将区分 "User" 的 "id"列和 "Address" 的 "id"列``Address``，其中``Address.id``渲染并跟踪为``id_1``：

.. sourcecode:: sql

  SELECT anon_1.id AS anon_1_id, anon_1.id_1 AS anon_1_id_1,
         anon_1.user_id AS anon_1_user_id,
         anon_1.email_address AS anon_1_email_address
  FROM (
    SELECT "user".id AS id, address.id AS id_1,
    address.user_id AS user_id, address.email_address AS email_address
    FROM "user" JOIN address ON "user".id = address.user_id
  ) AS anon_1 ORDER BY anon_1.email_address


:ticket:`5221`

从备选selectable选择实体; Query.select_entity_from()
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**简介**

:meth:`_orm.Query.select_entity_from`在2.0中将被删除::

    subquery = session.query(User).filter(User.id == 5).subquery()

    user = session.query(User).select_entity_from(subquery).first()

**2.0迁移**

如在:ref:`migration_20_query_from_self`中所述，:func:`_orm.aliased`对象提供了一个单一的地方，可以实现“从子查询选择实体”的操作。
使用:term:`1.x style`：

    from sqlalchemy.orm import aliased

    subquery = session.query(User).filter(User.name.like("%somename%")).subquery()

    ua = aliased(User, subquery)

    user = session.query(ua).order_by(ua.id).first()

使用:term:`2.0 style`：

    from sqlalchemy.orm import aliased

    subquery = select(User).where(User.name.like("%somename%")).subquery()

    ua = aliased(User, subquery)

    # 需要注意的是，如果需要的话，不会自动提供LIMIT 1
    user = session.execute(select(ua).order_by(ua.id).limit(1)).scalars().first()

**讨论**

这里的要点基本与在:ref:`migration_20_query_from_self`中讨论的相同。
:meth:`_orm.Query.select_from_entity`方法是指令查询从另一个可选择的实体加载一个特定ORM映射的实体的行，这要求ORM在稍后在查询中使用该实体时对其进行自动别名，例如 在“WHERE”或“ORDER BY”中。这个极其复杂的功能在这种情况下很少使用，就像:meth:`_orm.Query.from_self`的情况一样，使用显式的:func:`_orm.aliased`对象更容易跟随，无论从用户的角度还是从SQLAlchemy ORM的内部如何处理它的角度。

.. _joinedload_not_uniqued:

ORM行默认情况下不具有唯一性
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

**简介**

不再自动“唯一”的ORM行由``session.execute(stmt)``返回。这通常是一个受欢迎的变化，除了在使用“连接式急切加载”加载程序策略与集合的情况下::

    # 在旧API中，许多行都有相同的User主键，但是
    # 只有一个主键返回一个User
    users = session.query(User).options(joinedload(User.addresses))

    # 在新API中，唯一的优化可能不会自动地
    # 启用
    result = session.execute(select(User).options(joinedload(User.addresses)))

    # 这实际上会引发错误，让用户知道应该应用唯一性
    # rows
    rows = result.all()

**2.0迁移**

在使用集合的连接加载时，需要调用:meth:`_engine.Result.unique`方法。ORM实际上已经设置了默认的行处理程序，如果不执行此操作，它将引发错误，以确保联接急加载集合不返回重复行，同时仍然保持显式性：：

    # 1.4 / 2.0代码

    stmt = select(User).options(joinedload(User.addresses))

    # 由于集合的joinedload(),语句将引发唯一()，在所有其他情况下，不需要使用unique()。通过显式声明unique()，解决了查询到的对象数量/行数与“选择计算”的数量之间的混淆
    rows = session.execute(stmt).unique().all()

**讨论**

这种情况有点不寻常，因为SQLAlchemy要求调用一个它完全能够自动执行的方法。要调用该方法的原因是为了确保开发人员“选择”使用:meth:`_engine.Result.unique`方法，使得直接对行进行计数不冲突与实际结果集中的记录数，这是多年来用户混淆的长期来源和错误报告。默认情况下不会在任何其他情况下自动执行唯一性，这将提高性能，并在自动唯一性导致混淆的情况下提高清晰度。

就像在大多数情况下强烈建议使用:func:`_orm.selectinload`策略一样，在使用连接加载集合的情况下需要调用:meth:`_engine.Result.unique`方法，在 2.0中引入了以集合为导向的即时加载程序，这比:func:`_orm.joinedload`在大多数方面都要优越，并应予以优先考虑。

.. seealso::

    :ref:`change_7123`

    :ref:`write_only_relationship`

“动态”关系加载器被“只写”取代
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

🈚:ref:`dynamic_relationship`所述的“动态”关系加载程序使用的是:meth:`_query.Query`对象，在2.0中已经过时。 "动态"关系对协程的支持不直接兼容，此外，它不再实现防止迭代大型集合的原始目的，因为它具有几个隐式执行迭代的行为。

引入了称为“lazy=”write_only“” 的新的加载器策略，该新的策略通过:class:`_orm.WriteOnlyCollection`集合类提供了一个非常严格的“无隐式迭代”API，此外还与2.0样式语句执行集成，支持异步操作以及与新的:ref:`ORM-enabled Bulk DML <change_8360>`特性集成。

同时，“lazy=”dynamic“”仍然在2.0版中得到**完全支持**；在完全运行于2.0系列的应用程序上之前，可以推迟迁移此特定模式。

**迁移到2.0**

新的“写入”加载器策略仅在SQLAlchemy 2.0中提供，不是1.4的一部分。同时，“lazy=”dynamic“”加载器策略在2.0版中仍然得到完全支持，甚至包括新的pep-484和注释化映射支持。

因此，从“dynamic”进行迁移的最佳策略是**等待应用程序完全运行于2.0版**，然后直接从:class:`.AppenderQuery`(动态模式所使用的集合类型)迁移到:class:`.WriteOnlyCollection`(“write_only”策略所使用的集合类型)。

但是，在1.4中，也可以使用一些技术以更具“2.0”风格使用“lazy=”dynamic“”。
有两种方法可以实现特定关系的2.0样式查询：

1.在现有的“lazy=”dynamic“”关系上使用:attr:`_orm.Query.statement`属性。我们可以使用:meth:`_orm.Session.scalars`等方法直接使用动态加载程序，如下所示：

class User(Base):
    __tablename__ = "user"

    posts = relationship(Post, lazy="dynamic")


jack = session.get(User, 5)

# 过滤Jack的博客文章
posts = session.scalars(jack.posts.statement.where(Post.headline == "this is a post"))

2.使用:func:`_orm.with_parent`函数，在直接构造:func:`_sql.select`构造的情况下：

from sqlalchemy.orm import with_parent

jack = session.get(User, 5)

posts = session.scalars(
    select(Post)
    .where(with_parent(jack, User.posts))
    .where(Post.headline == "this is a post")
)

**讨论**

最初的想法是:func:`_orm.with_parent`函数应该就足够了，但是继续使用关系本身的特殊属性仍然很吸引人，并且没有理由一个2.0风格的构造不能在这里运行。

新的“只写”加载程序策略提供了一种新的集合类型，该类型不支持隐式迭代或项目访问。集合的内容是通过调用它的``.select()``方法来读取的以帮助构建合适的SELECT语句。该集合还包括``.insert()``, ``.update()``, ``.delete()``方法，可用于发出用于集合中的项目的批量DML语句。与“dynamic”功能类似，还有``.add()``, ``.add_all()``和``.remove()``方法，它们使用工作单元过程为添加或删除的单个成员排队。新功能的介绍方式为:ref:`change_7123`。

.. seealso::

    :ref:`change_7123`

    :ref:`write_only_relationship`


.. _migration_20_session_autocommit:

移除Session的自动提交模式; 增加自动开始支持
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`_orm.Session`不再支持“自动提交”模式，也就是这种模式::

    from sqlalchemy.orm import Session

    sess = Session(engine, autocommit=True)

    # 没有开始事务，但会发出SQL语句，将不再支持
    obj = sess.query(Class).first()


    # 会在Session闪存中产生begin并自动提交，将不再支持
    sess.flush()

**2.0迁移**

使用:class:`_orm.Session`的一个主要原因是使:meth:`_orm.Session.begin`方法可用，以便框架集成和事件钩子可以控制此事件。在1.4中，:class:`_orm.Session`现在特征：ref:_5074`，：meth:`_orm.Session.begin`方法可能现已调用::


    from sqlalchemy.orm import Session

    sess = Session(engine)

    sess.begin()  # 显式开始；如果未调用，将自动开始
    # 当需要数据库访问时

    sess.add(obj)

    sess.commit()

**讨论**

“自动提交”模式是SQLAlchemy第一个版本的一个遗留下来的东西。这个标志大部分保留下来，主要是为了允许显式使用:meth:`_orm.Session.begin`，这现在由1.4解决了，以及允许使用“子事务”的使用，这也在2.0中删除了。

.. _migration_20_session_subtransaction:

删除Session“子事务”行为
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

“子事务”模式经常与自动提交模式一起使用，也已在1.4中过时。此模式允许在已经开始事务的情况下，对:meth:`_orm.Session.begin`方法进行调用，从而产生一个称为“子事务”的构造，该构造本质上是一个阻止:meth:`_orm.Session.commit`方法实际提交的块。

**2.0迁移**

为了提供对使用此模式的应用程序的向后兼容性，可以使用以下上下文管理器，或基于装饰器的类似实现：


    import contextlib


    @contextlib.contextmanager
    def transaction(session):
        if not session.in_transaction():
            with session.begin():
                yield
        else:
            yield

上述上下文管理器可以像“子事务”标志一样使用，例如下面的示例所示：


    # method_a开始一个事务并调用method_b
    def method_a(session):
        with transaction(session):
            method_b(session)


    # method_b也开始一个事务，但是当
    #从method_a调用时参与正在进行的事务。
    def method_b(session):
        with transaction(session):
            session.add(SomeObject("bat", "lala"))


    Session = sessionmaker(engine)

    # 创建一个Session并调用method_a
    with Session() as session:
        method_a(session)

为了与首选的成语习惯相比较，将begin块放在最外层。这消除了各个函数或方法必须关注事务分界的细节的需要：：


    def method_a(session):
        method_b(session)


    def method_b(session):
        session.add(SomeObject("bat", "lala"))


    Session = sessionmaker(engine)

    # 创建一个Session并调用method_a
    with Session() as session:
        with session.begin():
            method_a(session)

**讨论**

这种模式在实际应用程序中被证明是困惑的，并且最好的做法是确保单个开头/提交对最高级别的数据库操作进行了执行。


2.0迁移 - ORM扩展和配方更改
------------------------------------------------

Dogpile缓存配方和水平切分使用新的会话API
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

随着:类:`_orm.Query`对象成为遗留API，这两个配方现在依赖于对:class:`_orm.SessionEvents.do_orm_execute`钩子方法的调用。参见:ref:`do_orm_execute_re_executing`中的示例。


烘焙查询扩展被内置缓存替换
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

烘焙查询扩展已被内置缓存系统替代，并且ORM内部不再使用它。

有关新缓存系统的全面背景信息，请参见:ref:`sql_caching`。


Asyncio支持
---------------------

SQLAlchemy 1.4包括Core和ORM的协程支持。
新API专门利用上述“future”模式。有关详情，参见:ref:`change_3414`。