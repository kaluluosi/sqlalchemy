配置版本计数器
=============================

:class:`_orm.Mapper`支持管理 :term:`版本ID列`，它是一个单个表列，每当映射表的“UPDATE”发生时增加或以其他方式更新其值。每次ORM发出“UPDATE”或“DELETE”来对行进行操作时都会检查该值，以确保内存中保存的值与数据库中的值匹配。

.. warning::

    由于版本控制功能依赖于对象的**内存**比较,因此该功能仅适用于:meth:`.Session.flush`过程，ORM将每个内存中的行刷新到数据库。在使用:meth:`_query.Query.update`或:meth:`_query.Query.delete`方法执行多行UPDATE或DELETE时，它不会生效，因为这些方法仅发出UPDATE或DELETE语句，但否则无法直接访问那些受影响的行的内容。

此功能的目的是检测并发事务在大致同时修改同一行的时间，或提供对在再次使用先前事务的数据而没有刷新的情况下使用“陈旧”行的系统的保护。 （例如，如果使用:class:`.Session`并设置了“expire_on_commit=False”，则可以从上一次事务重复使用数据）。

.. 主题:: 并发事务更新

    在检测事务内的并发更新时，通常情况下，数据库的事务隔离级别低于:term:`repeatable read`的级别;否则，事务将不会暴露为与本地更新值发生冲突的并发更新创建的新行值。在这种情况下，SQLAlchemy版本控制功能通常用于跨事务陈旧性检测，而不是用于事务内冲突检测。

    强制实施可重复读操作的数据库通常会锁定目标行以防止并发更新，或者会使用某种形式的多版本并发控制，以便在提交事务时发出错误。 SQLAlchemy的version_id_col则是一种允许特定表中的版本跟踪的替代方法，而在事务中可能没有设置此隔离级别。

    .. seealso::

        `可重复读取隔离级别 <https://www.postgresql.org/docs/current/static/transaction-iso.html#XACT-REPEATABLE-READ>`_ - PostgreSQL的可重复读取实现，包括错误条件说明。

简单版本计数
-----------------------

最简单的跟踪版本的方法是向映射表添加一个整数列，然后将其作为“version_id_col”建立在映射器选项中::

    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        version_id = mapped_column(Integer, nullable=False)
        name = mapped_column(String(50), nullable=False)

        __mapper_args__ = {"version_id_col": version_id}

.. 注意::  强烈建议将“version_id”列设置为NOT NULL。版本控制功能**不支持**版本控制列中的NULL值。

以上，``User``映射使用列``version_id``跟踪整数版本。 当首次刷新``User``类型的对象时，``version_id``列将被赋予值“1”。然后，以后的表更新将始终以类似于以下方式发出：

.. sourcecode:: sql

    UPDATE user SET version_id=:version_id, name=:name
    WHERE user.id = :user_id AND user.version_id = :user_version_id
    -- {"name": "new name", "version_id": 2, "user_id": 1, "user_version_id": 1}

上述UPDATE语句正在更新与"user.id = 1"匹配且还需要"user.version_id = 1"匹配的行，其中"1"是我们已知用于该对象的最后一个版本标识符。如果在其他地方的事务中独立修改了该行，则该版本ID将不再匹配，UPDATE语句将报告没有匹配的行; SQLAlchemy测试这种情况，即我们的UPDATE（或DELETE）语句只匹配一个行。如果匹配了零行，则表示我们的数据版本已过期，并引发:exc:`.StaleDataError`。

自定义版本计数器/类型
-------------------------------

还可以使用其他类型或计数器方案进行版本控制。常见类型包括日期和GUID。使用备用类型或计数器方案时，SQLAlchemy提供了一个使用“version_id_generator”参数的挂钩函数，该参数接受版本生成可调用项。此可调用项将传递当前已知版本的值，并且预计返回后续版本。

例如，如果我们想要使用随机生成的GUID跟踪“User”类的版本控制，我们可以这样做（请注意，某些后端支持原生GUID类型，但在这里我们使用简单的字符串来说明）：：

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

持久化引擎将在每次一个“User”对象需要执行INSERT或UPDATE时调用“uuid.uuid4()”。在这种情况下，我们的版本生成功能可以忽略“version”的输入值，因为“uuid4()”函数会生成没有任何先决条件值的标识符。如果我们正在使用数字或特殊字符系统等顺序版本控制方案，则可以利用给定的“version”来帮助确定后续值。

.. seealso::

    :ref:`custom_guid_type`

服务器端版本计数器
----------------------------

“version_id_generator”也可以配置以依赖于由数据库生成的值。在这种情况下，数据库需要一些生成新标识符的方法，当行受到INSERT和UPDATE的影响时需要这些方法。对于UPDATE情况，通常需要更新触发器，除非问题数据库支持某种本机版本标识符。 PostgreSQL数据库特别支持一个名为`xmin <https://www.postgresql.org/docs/current/static/ddl-system-columns.html>`_的系统列，提供UPDATE版本控制，我们可以使用PostgreSQL中的“xmin”列对“User”类进行版本控制，如下所示：：

    from sqlalchemy import FetchedValue


    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50), nullable=False)
        xmin = mapped_column("xmin", String, system=True, server_default=FetchedValue())

        __mapper_args__ = {"version_id_col": xmin, "version_id_generator": False}

通过上述映射，ORM将依赖于“xmin”列来自动提供版本ID计数器的新值。

.. 主题:: 创建引用系统列的表

    在上述情况下，“xmin”是由PostgreSQL提供的系统列，我们使用“system=True”参数将其标记为系统提供的列，从CREATE TABLE中省略。此列的数据类型是称为“xid”的内部PostgreSQL类型，其行为类似于字符串，因此我们使用:_types.String类型。

当ORM发出INSERT或UPDATE时，通常不会积极提取数据库生成的值，而是将这些列保留为“过期”并在下一次访问时获取，除非设置“eager_defaults”:_orm.Mapper标志。但是，当使用服务器端版本列时，ORM需要积极提取新生成的值。这样，版本计数器将在任何并发事务更新它之前设置。最好使用:term:`RETURNING`同时在INSERT或UPDATE语句中进行获取，否则如果之后发出SELECT语句，则仍存在潜在的竞争关系，其中版本计数器可能会在获取之前发生更改。

当目标数据库支持RETURNING时，我们的User类的INSERT语句如下：

.. sourcecode:: sql

    INSERT INTO "user" (name) VALUES (%(name)s) RETURNING "user".id, "user".xmin
    -- {'name': 'ed'}

在上述示例中，ORM可以在一条语句中获取任何新生成的主键值和服务器生成的版本标识符。当后端不支持RETURNING时，必须为**每个**INSERT和UPDATE发出额外的SELECT语句，这效率要低得多，并且还会引入可能的错误版本计数器:

.. sourcecode:: sql

    INSERT INTO "user" (name) VALUES (%(name)s)
    -- {'name': 'ed'}

    SELECT "user".version_id AS user_version_id FROM "user" where
    "user".id = :param_1
    -- {"param_1": 1}

强烈建议仅在绝对必要的情况下以及仅在支持:term:`RETURNING`的后端上使用服务器端版本计数器，当前为PostgreSQL、Oracle、MariaDB 10.5、SQLite 3.35和SQL Server。

编程或条件版本计数器
----------------------------------------------

当“version_id_generator”设置为False时，我们还可以以与分配任何其他映射属性相同的方式编程（并有条件地）设置我们对象上的版本标识符。例如，如果我们使用示例UUID，但将“version_id_generator”设置为“False”，我们可以按照我们的选择设置版本标识符：：

    import uuid


    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        version_uuid = mapped_column(String(32), nullable=False)
        name = mapped_column(String(50), nullable=False)

        __mapper_args__ = {"version_id_col": version_uuid, "version_id_generator": False}


    u1 = User(name="u1", version_uuid=uuid.uuid4())

    session.add(u1)

    session.commit()

    u1.name = "u2"
    u1.version_uuid = uuid.uuid4()

    session.commit()

我们还可以更新我们的“User”对象而不增加版本计数器。计数器的值将保持不变，UPDATE语句仍将检查以前的值。对于仅对并发问题敏感的UPDATE类别，这可能很有用：：

    # will leave version_uuid unchanged
    u1.name = "u3"
    session.commit()