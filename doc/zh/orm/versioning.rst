.. _mapper_version_counter:

配置版本计数器
=============================

  :class:`_orm.Mapper`  支持管理单个表列的版本 ID (version id column)，它在映射表更新时增加或更新它的值。每次 ORM 发出 ` `UPDATE`` 或 ``DELETE`` 命令时，都会检查此值，以确保内存中的值与数据库值匹配。

.. warning::

    因为版本控制特性依赖于比较对象的**内存中**记录，所以此特性仅适用于  :meth:`.Session.flush`  过程，ORM 将单个内存行刷新到数据库。但是，当对使用  :meth:` _query.Query.update`  或  :meth:`_query.Query.delete`  方法执行多行更新或删除时，它**不会**生效，因为这些方法仅发出更新或删除语句，但否则不直接访问受影响行的内容。

此特性的目的是检测当两个并行事务在大致相同时间修改同一行时，或者作为防范措施的 "过时" 行的使用，用于可以在不刷新的情况下重新使用从前一个事务获取的数据（例如，如果使用 ``expire_on_commit=False`` 的   :class:`.Session` ，则可能会重复使用来自以前事务的数据）。

.. topic:: 并发性事务更新

    在检测事务内并发更新时，通常情况下，数据库的事务隔离级别低于可重复读的级别；否则，事务将不会暴露给与本地更新值冲突的并发更新创建的新行值。在这种情况下，SQLAlchemy 版本控制特性通常无法用于事务内冲突检测，但仍可用于跨事务的过时检测。

    实施可重复读的数据库通常会锁定目标行以防止并发更新，或者使用某种类型的多版本并发控制，在提交事务时发出错误。 SQLAlchemy 的 version_id_col 是一种替代方案，它允许在事务中针对特定表进行版本跟踪，否则该表可能没有设置此隔离级别。

    .. seealso::

        `可重复读隔离级别 <https://www.postgresql.org/docs/current/static/transaction-iso.html#XACT-REPEATABLE-READ>`_ - PostgreSQL 的可重复读实现，包括错误条件的说明。

简单的版本计数
-----------------------

追踪版本最简单的方法是在映射表上添加一个整数列，然后将其作为 ``version_id_col`` 设为 mapper 选项的一部分：

    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        version_id = mapped_column(Integer, nullable=False)
        name = mapped_column(String(50), nullable=False)

        __mapper_args__ = {"version_id_col": version_id}

.. note:: 强烈建议将 "version_id" 列设置为 NOT NULL。此版本控制特性**不支持**未定义值。

上面的示例中，``User`` 映射使用 ``version_id`` 列跟踪整数版本。当首次刷新其类型为 ``User`` 的对象时，``version_id`` 列将被赋予值 "1"。之后，将总是以类似以下方式发送对表的 UPDATE：

.. sourcecode:: sql

    UPDATE user SET version_id=:version_id, name=:name
    WHERE user.id = :user_id AND user.version_id = :user_version_id
    -- {"name": "new name", "version_id": 2, "user_id": 1, "user_version_id": 1}

上述 UPDATE 语句正在更新与 ``user.id = 1`` 匹配且需要 ``user.version_id = 1`` 的行，其中 "1" 是我们已知在此对象上使用的最后一个版本标识符。如果此前在其他事务中修改了行，则此版本 ID 将不再匹配，UPDATE 语句将报告未匹配任何行。这是 SQLAlchemy 测试的条件，即我们的 UPDATE（或 DELETE）语句只匹配了一行。如果未匹配任何行，则表示我们的数据版本已过时，会引发  :exc:`.StaleDataError`  异常。

.. _custom_version_counter:

自定义版本计数器/类型
-------------------------------

可以使用其他类型或计数器方案进行版本控制。常见的类型包括日期和 GUID。在使用其他类型或计数器方案时，SQLAlchemy 提供了使用 ``version_id_generator`` 参数的钩子，接受版本生成可调用的方法。这个可调用方法会传递当前已知版本的值，并预期返回随后的版本值。

例如，如果我们想要使用随机生成的 GUID 跟踪 ``User`` 类的版本控制，我们可以这样做（注意，一些后端支持原生的 GUID 类型，但是在这里我们使用简单的字符串来说明）：

    import uuid



class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        version_uuid = mapped_column(String(32), nullable=False)
        name = mapped_column(String(50), nullable=False)

        __mapper_args__ = {
            "version_id_col": version_uuid,
            "version_id_generator": lambda version: uuid.uuid4().hex,
        }

每当对“User”对象进行INSERT或UPDATE操作时，持久性引擎都会调用``uuid.uuid4()``。在这种情况下，我们的版本生成函数可以忽略``version``的传入值，因为``uuid4()``函数会生成没有任何先决条件的标识符。如果我们使用顺序版本控制方案，例如数字或特殊字符系统，则可以利用给定的``version``来帮助确定后续值。

可参见::

      :ref:`custom_guid_type` 

.. _server_side_version_counter:

服务器端版本计数器
----------------------------

``version_id_generator``也可以配置为依赖于由数据库生成的值。在这种情况下，当对行进行INSERT或UPDATE操作时，数据库需要某种生成新标识符的方法。对于UPDATE情况，通常需要一个更新触发器，除非所涉及的数据库支持其他本机版本标识符。尤其是PostgreSQL数据库支持一个称为“xmin”<https://www.postgresql.org/docs/current/static/ddl-system-columns.html>`_的系统列，提供UPDATE版本控制。我们可以使用PostgreSQL的``xmin``列按以下方式对``User``类进行版本化：

    from sqlalchemy import FetchedValue


    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50), nullable=False)
        xmin = mapped_column("xmin", String, system=True, server_default=FetchedValue())

        __mapper_args__ = {"version_id_col": xmin, "version_id_generator": False}

通过以上映射，ORM将依靠``xmin``列自动提供版本ID计数器的新值。

.. topic:: 创建引用系统列的表格

    在上述方案中，由于``xmin``是PostgreSQL提供的系统列，我们使用``system=True``参数，将其标记为系统提供的列，从``CREATE TABLE``语句中省略。此列的数据类型是一个名为``xid``的内部PostgreSQL类型，它的作用大多像一个字符串，因此我们使用 :class:`_types.String` 数据类型。

当ORM发出INSERT或UPDATE时，通常不会主动提取数据库生成的值，而是将这些列保留为“过期”，在下一次访问它们时再提取，除非设置了``eager_defaults``   :class:`_orm.Mapper`  一次性地在INSERT或UPDATE语句中同时进行此提取，否则，如果之后再发出SELECT语句，则仍然存在可能的竞争条件，版本计数器可能在其被提取之前发生变化。

当目标数据库支持RETURNING时，我们``User``类的INSERT语句如下所示：

.. sourcecode:: sql

    INSERT INTO "user" (name) VALUES (%(name)s) RETURNING "user".id, "user".xmin
    -- {'name': 'ed'}

上述ORM可以在一条语句中获取任何新生成的主键值以及服务器生成的版本标识符。当后端不支持RETURNING时，必须为**每个**INSERT和UPDATE发出额外的SELECT语句，这样效率要低得多，而且还会引入版本计数器丢失的可能性：

.. sourcecode:: sql

    INSERT INTO "user" (name) VALUES (%(name)s)
    -- {'name': 'ed'}

    SELECT "user".version_id AS user_version_id FROM "user" where
    "user".id = :param_1
    -- {"param_1": 1}

非常强烈建议只在绝对必要且仅在支持:returning的后端上使用服务器端版本计数器，目前支持此功能的后端为PostgreSQL，Oracle，MariaDB 10.5，SQLite 3.35和SQL Server。

编程或条件版本计数器
--------------------------------------------

当``version_id_generator``设置为``False``时，我们也可以像分配任何其他映射属性一样在对象上以编程方式(并有条件地)设置版本标识符。例如，如果我们使用UUID示例，但将``version_id_generator``设置为``False``，那么我们可以在任意时刻设置版本标识符:

    import uuid


    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        version_uuid = mapped_column(String(32), nullable=False)
        name = mapped_column(String(50), nullable=False)我们也可以更新我们的``User``对象而不增加版本计数器；计数器的值将保持不变，并且UPDATE语句仍将根据先前的值进行检查。这可能对只有特定类别的UPDATE敏感于并发问题的方案有用::

    # 将保持version_uuid不变
    u1.name = "u3"
    session.commit()