=============
2.0 变更日志
=============

.. changelog_imports::

    .. include:: changelog_14.rst
        :start-line: 5


.. changelog::
    :version: 2.0.20
    :include_notes_from: unreleased_20

.. changelog::
    :version: 2.0.19
    :released: 2023年7月15日

    .. change::
        :tags: bug, orm
        :tickets: 10089

        修复了将关系集合直接设置为新集合，而新集合中的对象已经存在时，不会触发该对象的级联事件，
        导致如果该对象不存在，则无法将其添加到   :class:`_orm.Session`  中的问题。
        这与  :ticket:`6471`  类似，并且由于在 2.0 系列中移除了` `cascade_backrefs``，因此这个问题更加明显。
        作为  :ticket:`6471`  的一部分添加的事件  :meth:` _orm.AttributeEvents.append_wo_mutation`  现在也会针对已经存在于相同集合的批量设置的那个相同集合的成员发出。

    .. change::
        :tags: bug, engine
        :tickets: 10093

        将  :attr:`_result.Row.t`  和  :meth:` _result.Row.tuple`  重命名为
         :attr:`_result.Row._t`  和  :meth:` _result.Row._tuple` ；这是为了遵循 Python 标准库 ``namedtuple`` 的所有方法和预定义属性均以下划线开头的风格，
        以避免与现有列名称冲突。之前的方法/属性现已弃用，并将发出弃用警告。

    .. change::
        :tags: bug, postgresql
        :tickets: 10069

        修复了在  :ticket:`10004`  中对 PostgreSQL URL 解析的改进引起的回归，此时 query 字符串参数中有冒号，
        以支持各种第三方代理服务器和/或方言，由于这些被解释为 ``host:port`` 组合，因此无法正确解析。解析已更新，
        仅当主机名仅包含带点或破折号的字母数字字符（例如没有斜杠），后跟一个包含零个或多个整数的全整数令牌时，
        才将冒号视为仅指示 ``host:port`` 值。在所有其他情况下，将整个字符串视为主机。

    .. change::
        :tags: bug, engine
        :tickets: 10079

        对   :func:`_engine.make_url`  函数添加了对非字符串和非   :class:` _engine.URL`  对象的检测，
        允许立即引发 ``ArgumentError``，而不是稍后导致的失败。特殊逻辑确保允许   :class:`_engine.URL` 
        的模拟表单通过。贡献者：Grigoriev Semyon。

    .. change::
        :tags: bug, orm
        :tickets: 10090

        修复了通过反向引用与未加载的集合关联的对象，但由于在 2.0 系列中移除了``cascade_backrefs``，
        而未合并到   :class:`_orm.Session`  中的对象不会发出警告，即使它们是集合的待定成员并且应发出这些对象未包括在刷新中的警告的情况；在其他这样的情况下，
        当正在刷新的集合包含将被基本舍弃的非附加对象时，会发出警告。为反向引用 - 等待中的集合成员添加警告，
        以建立与可能在不同时间基于不同的关系加载策略出现或不存在和可能被刷新或未刷新的不同时间存在或不存在的集合的更大一致性。

    .. change::
        :tags: bug, postgresql
        :tickets: 10096

        修复了将   :class:`_postgresql.CITEXT`  数据类型与 ` `VARCHAR`` 类型进行比较的问题，
        导致右侧不被解释为 ``CITEXT`` 数据类型，适用于 asyncpg、psycopg3 和 pg80000 方言。
        这导致：class:`_postgresql.CITEXT` 类型在实用上不可用；现在已经修复，测试套件已经纠正，以正确断言表达式被正确呈现。

    .. change::
        :tags: bug, orm, regression
        :tickets: 10098

        修复了  :ticket:`9805`  引起的另一个回归，其中更激进的 "ORM" 标志传播会导致在将 ORM   :class:` .Query`  构造（尽管其中不包含 ORM 实体）嵌入到核心 SQL 语句中时，可以导致内部属性错误的问题。
        在此情况下，ORM 标志会导致错误的 Dataclass 默认值被直接分配给新实例，绕过了  :paramref:`_schema.Column.default`  取值为默认生成器时的默认生成器发生的默认生成器。
        现在检测到此情况，以维护以前的行为，但发出用于此类二义性用途的弃用警告；
        要填充   :class:`_schema.Column`  的默认生成器  :paramref:` _orm.mapped_column.default` ，应使用  :paramref:`_orm.mapped_column.insert_default`  参数，
        其名称按照 pep-681 是固定的  :paramref:`_orm.mapped_column.default` .

.. changelog::
    :version: 2.0.18
    :released: 2023年7月5日

    .. change::
        :tags: usecase, typing
        :tickets: 10054

        在使用 ``sqlalchemy.sql.operators`` 中的独立的运算符函数（比如 ``sqlalchemy.sql.operators.eq``）时，改进了类型。

    .. change::
        :tags: usecase, mariadb, reflection
        :tickets: 10028

        允许从 MariaDB 反映   :class:`_types.UUID`  列。这允许 Alembic 正确检测现有 MariaDB 数据库中的该列的类型。

    .. change::
        :tags: bug, postgresql
        :tickets: 9945

        将新参数 ``native_inet_types=False`` 添加到所有 PostgreSQL 方言中，
        它指示 DBAPI 中用于将 PostgreSQL   :class:`.INET`  和   :class:` .CIDR`  列转换为 Python ``ipaddress`` 数据类型的转换器应禁用，
        从而返回字符串。这允许编写使用字符串来处理这些数据类型的代码无需更改代码即可迁移到 asyncpg、psycopg 或 pg8000。
        .. seealso::
              :ref:`postgresql_network_datatypes` 

    .. change::
        :tags: usecase, extensions
        :tickets: 10013

        将新选项添加到   :func:`.association_proxy`  中：  :paramref:` .association_proxy.create_on_none_assignment`  ；
        当一个仅引用标量关系的关联代理分配值 ``None``，且未出现参考对象时，将通过创建器创建新对象。
        这显然是 1.2 系列中未定义的行为，现已被默默删除。

    .. change::
        :tags: bug, typing
        :tickets: 10061

        修复了   :func:`_orm.aliased`  构造中的一些类型问题，以正确接受使用  :meth:` .Table.alias`  别名的   :class:`.Table`  对象，以及支持一般的   :class:` .FromClause`  对象作为 "selectable" 参数，因为这都受支持。

    .. change::
        :tags: bug, engine
        :tickets: 10025

        调整了  :paramref:`_sa.create_engine.schema_translate_map`  功能，以便**所有**语句中的模式名称都被标记，而不管该名称是否在给定的实际模式转换映射中，
        并在执行时回退到使用原始名称时，当实际模式转换映射中不存在该密钥时。这两个更改允许在每次运行时使用具有已包括或未包括不同键集的模式翻译映射编译对象，
        使得当每次使用不同的键集的模式转换映射时，缓存的 SQL 构造仍然可以在运行时继续运行。此外，增加了对针对同一语句从同一位置调用时获得或失去空值键的 schema_translate_map 字典的检测，这会影响语句的编译，并且与缓存不兼容，将为这些情况引发异常。

    .. change::
        :tags: bug, mssql, sql
        :tickets: 9932

        修复了使用显式排序顺序的字符串类型   :class:`.Cast` （即带有：attr:` .Cast.collation` 参数）时，
        将 COLLATE 子句呈现在 CAST 函数内部的问题，导致语法错误。

    .. change::
        :tags: usecase, mssql
        :tickets: 7340

        在 MSSQL 方言中添加了创建和反射 COLUMNSTORE 索引的支持。
        可以在指定 ``mssql_columnstore=True`` 的索引上指定。

    .. change::
        :tags: usecase, postgresql
        :tickets: 10004

        为 asyncpg 方言添加了支持多个主机。对 PostgreSQL URL 例程进行了一般改进和错误检测，以支持“多主机”用例的添加。
        贡献者：Ilia Dmitriev。
        .. seealso::
              :ref:`asyncpg_multihost` 

.. changelog::
    :version: 2.0.17
    :released: 2023年6月23日

    .. change::
        :tags: usecase, postgresql
        :tickets: 9965

        pg8000 方言现在支持 RANGE 和 MULTIRANGE 数据类型，使用   :ref:`postgresql_ranges`  描述的现有 RANGE API。
        范围和多范围类型在版本为 1.29.8 之后的 pg8000 驱动程序中受支持。
        贡献者：Tony Locke。

    .. change::
        :tags: bug, orm, regression
        :tickets: 9870

        修复了 2.0 系列中的回归，其中使用   :func:`.undefer_group`  和   :func:` _orm.selectinload`  或   :func:`_orm.subqueryload`  的查询将引发 ` `AttributeError``。 
        贡献者：Matthew Martin。

    .. change::
        :tags: bug, orm
        :tickets: 9957

        修复了 ORM Annotated Declarative 中的问题，在其中声明的属性由未返回   :class:`.Mapped`  数据类型的 mixin 使用时，将返回错误。
        Declarative 运行时会错误地尝试将此注释解释为需要   :class:`.Mapped`  并引发错误。

    .. change::
        :tags: bug, orm, typing
        :tickets: 9957

        修复了类型问题，其中使用   :class:`.AssociationProxy`  返回类型从   :class:` _orm.declared_attr`  函数中无法使用完全。

    .. change::
        :tags: bug, orm, regression
        :tickets: 9936

        修复了 2.0.16 版中由  :ticket:`9879`  引入的回归，其中在   :class:` _orm.mapped_column`  的  :paramref:`_orm.mapped_column.default`  参数中传递可调用函数，
        而同时设置 ``init=False``，将会将此值解释为 Dataclass 默认值，该值将直接分配给对象的新实例，绕过底层   :class:`_schema.Column`  上  :paramref:` _schema.Column.default`  值生成器发生的默认生成器。 

        现在检测到这种情况，以维护以前的行为，但发出用于此类二义性用途的弃用警告。
        要填充   :class:`_schema.Column`  的默认生成器，应使用  :paramref:` _orm.mapped_column.insert_default`  参数，这将从  :paramref:`_orm.mapped_column.default`  参数中的
        固定  :meth:`_schema.Column.default_factory`  回调函数，该名称按照 pep-681 是固定的。

    .. change::
        :tags: bug, orm
        :tickets: 9777

        修改了 ``JoinedLoader`` 实现的某个特定区域的路径问题，在这个区域，它之前使用了一个会在线程之间共享的缓存结构，现在为了避免多个重复的操作条件而使用了一种更简单的方法。
        这个产品逻辑是为了避免多个重复的操作条件，并疑似造成了多次报告的一个在使用中的崩溃的原因。该缓存结构最终仍通过编译的 SQL 缓存进行“缓存”，因此不预期会降低性能。

    .. change::
        :tags: bug, orm, dataclasses
        :tickets: 9879

        修复了在   :class:`_schema.ForeignKey` （或其他列级约束）中使用   :class:` _orm.mapped_column`  的问题，
        该约束随后通过 pep-593   :class:`~typing.Annotated`  把它复制到模型中时，将其复制到目标   :class:` _schema.Table`  中产生了重复的每个约束，
        导致 CREATE TABLE DDL 的不正确以及在 Alembic 下的迁移指令。

    .. change::
        :tags: bug, orm
        :tickets: 9779

        修复了使用   :func:`_orm.joinedload`  加载选项的附加关系标准，并且这些附加关系标准本身包含相对于已加入实体的子查询的相关子查询和因此也需要对别名实体进行“调整” 所需的关联 Entity 的情况，在这种情况下，将被排除在此适应性之外，从而导致连接负载的 ON 子句错误。

    .. change::
        :tags: bug, postgresql
        :tickets: 9836

        使用对 PostgreSQL 特定运算符的正确优先级，例如 ``@>``。以前，优先级不正确，导致在针对 ``ANY`` 或 ``ALL`` 构造子句呈现时产生不正确的括号。

    .. change::
        :tags: bug, orm
        :tickets: 9869

        在  :meth:`_orm.registry.map_imperatively`  方法的  :paramref:` _orm.registry.map_imperatively.local_table`  参数检查中进行了改进，
        确保只传递   :class:`.Table`  或其它   :class:` .FromClause` ，而不要传递现有映射类，因为会进一步解释该对象以进行新的映射。

.. changelog::
    :version: 2.0.16
    :released: 2023年5月19日

    .. change::
        :tags: bug, sql
        :tickets: 9772

        修复了   :func:`_sql.values`  构造在标量子查询中使用时会导致内部编译错误的问题。

    .. change::
        :tags: usecase, sql
        :tickets: 9752


        将 MSSQL 的   :func:`_sql.try_cast`  函数改为了泛数据，这意味着它可能由第三方方言实现。
        在 SQLAlchemy 中，  :func:`_sql.try_cast`  函数仍然是仅适用于 SQL Server 的一个结构，
        如果在后端上不支持它，它将会引发   :class:`.CompileError` 。  :func:` _sql.try_cast`  实现了一个 CAST，
        其中不能转换的转换将返回 NULL，而不是引发错误。从理论上讲，第三方方言可实现该结构用于 Google BigQuery、DuckDB 和 Snowflake 等平台，可能还有其他平台。
        贡献者：Nick Crews。

    .. change::
        :tags: bug, tests, pypy
        :tickets: 9789

        修复了依赖于 ``sys.getsizeof()`` 函数的测试在 pypy 上无法运行的问题，
        在 pypy 上，该函数似乎与在 cpython 上的行为不同。

    .. change::
        :tags: bug, orm
        :tickets: 9777

        修改了 ``JoinedLoader`` 实现的某个特定区域的路径问题，在这个区域，它之前使用了一个会在线程之间共享的缓存结构，
        现在为了避免多个重复的操作条件而使用了一种更简单的方法。该产品逻辑是为了避免多个重复的操作条件，并疑似造成了多次报告的一个在使用中的崩溃的原因。
        该缓存结构最终仍通过编译的 SQL 缓存进行“缓存”，因此不预期会降低性能。 

    .. change::
        :tags: bug, orm, regression
        :tickets: 9767

        修复了在   :class:`_sql.CTE`  构造中使用   :func:` _dml.update`  或   :func:`_dml.delete` ，然后在   :func:` _sql.select`  中使用会导致引发   :class:`.CompileError`  的 ORM 相关规则的问题。

    .. change::
        :tags: bug, orm
        :tickets: 9766

        修复了新的 ORM Annotated Declarative 中的问题，在其中使用列级约束，例如   :class:`_schema.ForeignKey` ，并使用   :func:` _orm.mapped_column` ，然后通过 pep-593
        在数据模型中将其复制出来，将产生重复的每个约束，导致 CREATE TABLE DDL 不正确以及在 Alembic 下的迁移指令。

    .. change::
        :tags: usecase, postgresql
        :tickets: 9041

        合并 PostgreSQL 自定义操作符定义，因为它们在多个不同数据类型之间共享。

    .. change::
        :tags: bug, orm
        :tickets: 9913

         :attr:`_orm.InstanceState.unloaded_expirable`  属性是  :attr:` _orm.InstanceState.unloaded`  的同义词，现在已废弃；此属性始终是特定于实现的，不应公开。

    .. change::
        :tags: usecase, postgresql
        :tickets: 8240

        为 PostgreSQL 10 的唯一索引和唯一约束添加了支持，使用方言选项 ``postgresql_nulls_not_distinct``。
        还更新了反射逻辑，以正确考虑此选项。
        贡献者：Pavel Siarchenia。

.. changelog::
    :version: 2.0.15
    :released: 2023年5月10日

    .. change::
        :tags: usecase, asyncio
        :tickets: 9731

        添加了一个新的辅助 mixin   :class:`_asyncio.AsyncAttrs` ，
        它旨在提高懒加载器和其他过期或延迟 ORM 属性与 asyncio 的使用情况，
        提供了一个简单的属性访问器，它为任何 ORM 属性提供一个“await”的接口，无论是否需要发出 SQL。

        .. seealso::

              :class:`_asyncio.AsyncAttrs` 

    .. change::
        :tags: bug, orm
        :tickets: 9717

        修复了 ORM Annotated Declarative 中的问题，在其中使用 ``from __future__ import annotations``，与 Pydantic dataclasses 结合使用时。

    .. change::
        :tags: typing, sql
        :tickets: 9656

        添加了公共类型  :data:`_sql.ColumnExpressionArgument` ，用于表示传递给 SQLAlchemy 构造的基于列的参数，例如  :meth:` _sql.Select.where` 、  :func:`_sql.and_`  等。
        这可用于向调用这些方法的终端用户函数添加类型。

    .. change::
        :tags: bug, orm
        :tickets: 9746

        修复了在   :ref:`orm_queryguide_upsert_returning`  中使用的问题，此问题中未将 ` `populate_existing`` 执行选项传播到加载选项，从而防止刷新现有属性。

    .. change::
        :tags: bug, sql

        修复了特定的错误，由于 Oracle   :class:`_oracle.BINARY_DOUBLE`  现在子类化   :class:` _sqltypes.Double` ，
        而   :class:`_sqltypes.Float`  的内部类型用于执行特定于 dialect 的 float/double 类型的 PG8000 和 Asyncpg 的内部类型现在正确地子类化了   :class:` _sqltypes.Float` 。

    .. change::
        :tags: bug, ext
        :tickets: 9676

        修复了   :class:`_mutable.Mutable`  中的问题，其中 ORM 映射的属性的事件注册将针对多个不同的映射继承子类重复调用，导致在继承层次结构中调用重复事件。

    .. change::
        :tags: bug, orm
        :tickets: 9715

        修复了加载器策略路径问题，其中，对于基于   :func:`_orm.with_polymorphic`  或类似方法的中间成员但 IDM loader 是必需的的多级别深度的跟随，急切加载器（例如   :func:` _orm.joinedload`  /   :func:`_orm.selectinload`  ）在很多情况下会失败。

    .. change::
        :tags: usecase, sql
        :tickets: 9721

        实现了 UPDATE 和 DELETE 语句的“笛卡尔积警告”，其中包括多个未以某种方式相互关联的表。

    .. change::
        :tags: bug, sql

        修复了   :func:`_dml.update`  构造中包含多个表且没有值子句时会引发内部错误的问题。
        没有值的   :class:`_dml.Update`  的当前行为是生成具有空的“set”子句的 SQL UPDATE 语句，因此对于此特定子案例已被建立。

    .. change::
        :tags: bug, postgresql
        :tickets: 9773

        修复了很久以前的问题，当  :paramref:`_postgresql.ENUM.create_type`  设置为其非默认值 ` `False`` 时，
        它将不会在复制列时进行传播，这是通过 ORM Declarative 混合常见的。
此版本只包括小改动，没有具体更改的记录。该版本修复了许多bug，包括ORM、SQL、异步、MySQL等方面的修复。具体包括以下几点：

- 在ORM的内部专用模块`EvaluatorCompiler`中添加私有修饰符，避免了不希望被使用的bug。
- 改进了ORM映射，增加了一个新的事件钩子`after_mapper_constructed`，并且更改了一些属性的默认行为。
- 将所有二进制位操作符添加为新的SQL函数。
- 修复了MySQL 8的问题，支持新的“AS <name> ON DUPLICATE KEY”语法。
- 修复了ORM声明性映射中的一些问题，包括与混合使用的问题、`primary_key`参数在`mapper_args`中使用的问题等等。

其他一些小问题和早期版本中的错误也已纠正。.. _bugs-2.0.0rc4:

.. _change_9164:

修复了 ORM 模型中使用复合外键的 joined table 继承遇到 Mapper 内部错误的回归问题。 (  :ref:`ticket_9164` ) 

.. _change_7664:

修正了版本2.0.0中  :ticket:`7664`  的修复，以包括无意中遗漏的   :class:` .DropSchema` ，在这个修复中，允许在没有方言的情况下进行字符串化。这两个修复已被移植到1.4.47。 (  :ref:`ticket_7664` ) 

.. _change_9175:

添加支持  :pep:`484`  中的 ` `NewType`` 用于  :paramref:`_orm.registry.type_annotation_map`  和   :class:` .Mapped`  构造中。这些类型的行为与当前操作的类型自定义子类相同；它们必须显式出现在  :paramref:`_orm.registry.type_annotation_map`  中以进行映射。(  :ref:` ticket_9175` ) 

.. _change_9183:

修复了 limit/offset 方法中的类型错误，包括  :meth:`.Select.limit` ,  :meth:` .Select.offset` ,  :meth:`_orm.Query.limit`  和  :meth:` _orm.Query.offset` ，以允许 ``None`` 这是取消当前的limit/offset。(  :ref:`ticket_9183` ) 

.. _change_9179:

当使用   :class:`.MappedAsDataclass`  超类时，在层级结构内所有的类都是其子类时，无论是否实际映射，都将通过 ` `@dataclasses.dataclass`` 方法运行。这意味着，在映射的子类转换为数据类时，层级内部未映射的非 ORM 字段也将被使用。此行为适用于使用 ``__abstract__ = True`` 进行映射的中间类，以及适用于用户定义的声明性基类本身，假设类中存在   :class:`.MappedAsDataclass`  作为这些类的超类。(  :ref:` ticket_9179` ) 

.. _change_9170:

修复了一个类型错误，使得被类型标记为   :class:`_orm.Mapped`  的   :func:` _orm.mapped_column`  对象不能被接受为模式约束，如   :class:`_schema.ForeignKey` 、  :class:` _schema.UniqueConstraint`  或   :class:`_schema.Index` 。(  :ref:` ticket_9170` ) 

.. _change_9200:

修复了   :class:`.DeclarativeBase`  类中的回归问题，其中注册表的默认构造函数不会被应用于基类本身，这与之前的   :func:` _orm.declarative_base`  构造方式不同。这将阻止具有自己的 ``__init__()`` 方法的映射类调用 ``super().__init__()`` 以访问注册表的默认构造函数并自动填充属性，而是回到了 ``object.__init__()``，这将在任何参数上引发错误。(  :ref:`ticket_9171` ) 

.. _change_9173:

针对新的“insertmanyvalues”功能的实现引起的回归问题进行修复，其中在 CTE 中通过   :func:`_sql.insert`  参考另一个   :func:` _sql.insert`  的情况下会发生内部 ``TypeError``，在使用“insertmanyvalues” 时，针对位置方言如 asyncpg 的这种使用情况进行了进一步修复。(  :ref:`ticket_9173` ) 

.. _change_9156:

修复了  :meth:`_expression.ColumnElement.cast`  的类型注释, 使其可以接受 ` `Type[TypeEngine[T]]`` 和 ``TypeEngine[T]``。以前只接受 ``TypeEngine[T]``。 (  :ref:`ticket_9156` ) 

.. _change_9187:

添加了支持  :pep:`586`  中 ` `Literal[]`` 的能力，用于  :paramref:`_orm.registry.type_annotation_map`  以及在   :class:` .Mapped`  构造中使用。要使用此类自定义类型，它们必须显式出现在  :paramref:`_orm.registry.type_annotation_map`  中以进行映射。来源于 Frederik Aalund 的拉取请求。

作为此更改的一部分，对  :paramref:`_orm.registry.type_annotation_map`  中的   :class:` .sqltypes.Enum`  的支持已扩展，以包括支持包含字符串值的 ``Literal[]`` 类型的用法，作为 ``enum.Enum`` 数据类型的补充。如果在 ``Mapped[]`` 中使用了未在  :paramref:`_orm.registry.type_annotation_map`  中链接到特定数据类型的 ` `Literal[]`` 数据类型，则会默认使用   :class:`.sqltypes.Enum` 。

.. seealso::

      :ref:`orm_declarative_mapped_column_enums` 

(  :ref:`ticket_9187` ) 

.. _change_9182:

改进了在将策略选项从一个基类链接到子类的另一个属性时报告错误的方式，其中应使用 ``of_type()``。以前，当使用  :meth:`.Load.options`  时，消息缺乏详细信息，即应使用 ` `of_type()``，而在直接链接选项时不是这种情况。现在，即使使用  :meth:`.Load.options`  时，也会发出详细的信息。

(  :ref:`ticket_9182` ) 

.. _change_9183:

修复了 limit/offset 方法的类型错误，包括  :meth:`.Select.limit` ,  :meth:` .Select.offset` ,  :meth:`_orm.Query.limit`  和  :meth:` _orm.Query.offset` , 以允许使用 ``None`` 是已记录的 API，以“取消”当前的limit/offset。

(  :ref:`ticket_9183` ) 

.. _change_9175:

添加了 ``NewType`` 的支持，可用于  :paramref:`_orm.registry.type_annotation_map`  和   :class:` .Mapped`  结构中，使其行为与当前操作的自定义类型子类相同。在映射这些类型时，它们必须出现在  :paramref:`_orm.registry.type_annotation_map`  中。

(  :ref:`ticket_9175` ) 

.. _change_9164:

修复了 ORM 模型中使用复合外键的 joined table 继承遇到 Mapper 内部错误的回归问题。

(  :ref:`ticket_9164` ) 

.. _change_7664:

修正在版本2.0.0中出现的  :ticket:`7664`  的修复，以包括无意中遗漏的   :class:` .DropSchema` ，在这个修复中，允许在没有方言的情况下进行字符串化。这两个修复已被移植到1.4.47。

(  :ref:`ticket_7664` ) 

.. _change_9025:

对   :class:`_orm.Mapper`  添加了新的  :paramref:` _orm.Mapper.polymorphic_abstract`  指令，因此 ORM 现在不会考虑直接实例化或加载类本身，只考虑子类。

实际上，使用  :paramref:`_orm.Mapper.polymorphic_abstract`  的类可以用作   :func:` _orm.relationship`  的目标，以及用于查询；子类必须在映射中包含多态身份。

(  :ref:`ticket_9025` ).. 将rst文档翻译成中文

.. changelog::
    :version: 2.0.0b4
    :released: January 26, 2023
    :released: December 5, 2022

    .. change::
        :tags: bug, sql
        :tickets: 8994

        为了适应有不同字符转义需要的第三方方言，通过使用可重写的  :attr:`.SQLCompiler.bindname_escape_chars`   字典，系统"转义"（即将特殊字符替换为另一个字符），将特殊字符的参数名字部分适合第三方方言。此更改还添加了点"."作为默认转义字符。


    .. change::
        :tags: orm, feature
        :tickets: 8889

        为  :paramref:`.Mapper.eager_defaults`  参数添加了一个新的默认值"auto"。在一次工作单元的 flush 过程中，如果方言支持 INSERT 的 RETURNING，并且有   :ref:` insertmanyvalues <engine_insertmanyvalues>`  可用，将自动获取表格的默认值。如果设置  :paramref:`.Mapper.eager_defaults`  为 "True"，则在表示服务器端 UPDATE 默认值（非常不常见）的情况下，持续取用即可；而对于 UPDATE 语句没有批量 RETURNING，不会因为eager_defaults设置为True而触发对默认值的加载。

    .. change::
        :tags: usecase, orm
        :tickets: 8973

        当在非 `Mapped[]` 注释中检测到非 `Mapped[]` 注释中的注释(例如使用`(a: type)`验证构造器)时，现在不再需要使用 `__allow_unmapped__`属性来标记 `DeclarativeDataclass Mapped` 类。原来的错误以不会提供有关如何针对Dataclasses的实际模式的正确声明解释。现在，如果使用  :meth:`_orm.registry.mapped_as_dataclass`  或   :class:` _orm.MappedAsDataclass` ，则不再引发此错误消息。

    .. change::
        :tags: bug, orm
        :tickets: 8812

        当刷新映射到子查询的映射类（例如直接映射或某些形式的具体表继承）时，使用 :paramref:`_orm.Mapper.eager_defaults` 参数时会导致失败，之前可能会出现错误。现已修复此问题。


.. changelog::
    :version: 2.0.0b3
    :released: January 26, 2023
    :released: November 4, 2022

    .. change::
        :tags: bug, orm, declarative
        :tickets: 8759

        为   :class:`.Mapped`  注释添加了 ORM 的支持，支持关联的另一个类名作为第一个参数传入。添加了另一种方法，如果类名没有导入可以使用，即使用关联的 `  :class:` _orm.Mapped` ` 符号的名称作为类名。另外，指定为   :func:`_orm.relationship`  的主参数，传递的类名也将始终优先于注释中给出的名称（因为其它名称可能无法导入）。


    .. change::
        :tags: bug, orm
        :tickets: 8692

        改善了在注释中使用不包括 ``Mapped[]`` 的注释的旧 1.4 映射的支持，通过确保 `__allow_unmapped__` 属性可以使用，始终会使这种旧注释通过 Annotated Declarative 而不激发错误 。此外，改进了检测到此情况时生成的错误消息，并增加了有关如何处理此情况的更多文档。不幸的是，1.4 WARN_SQLALCHEMY_20 迁移警告 **不能** 检测到这种情况。


    .. change::
        :tags: usecase, postgresql
        :tickets: 8690

        优化了在  :ref:`change_7156` 中描述的新 PostgreSQL   :class:` .Range`  对象的处理方式，以适应特定于驱动程序的范围和多范围对象，以更好地适应遗留代码和从原始 SQL 结果集传回新范围或多范围表达式的情况。

    .. change::
        :tags: usecase, engine
        :tickets: 8717

        在  :meth:`.PoolEvents.reset`  事件中添加了新参数  :paramref:` .PoolEvents.reset.reset_state` ，并在其中设置了包含有关如何重置连接的各种状态信息，以允许具有完整上下文的自定义重置计划运行。在此更改中还包括修复，在 1.4 版本中重新启用  :meth:`.PoolEvents.reset`  事件，使其在所有情况下都能运行，包括   :class:` .Connection`  已经 "重置" 连接。这两个变化共同允许使用  :meth:`.PoolEvents.reset`  事件来实现自定义重置计划，而不是  :meth:` .PoolEvents.checkin`  事件。


    .. change::
        :tags: bug, orm, declarative
        :tickets: 8705

        修改了   :class:`.Mapper`  的基本配置行为，此前当  :paramref:` _orm.Mapper.properties`  字典中明确存在   :class:`_schema.Column`  对象时，无论是直接还是封装在映射器属性对象中，它们都将在映射到可映射   :class:` .Table` （或其他可选择性）本身（前提是它们实际上属于该表的列列表）中，保持所映射的   :class:`.Table`  列的相同顺序，从而保持分配到映射类的属性的顺序与映射类上的封装以及在 ORM SELECT 语句中呈现的属性的顺序一致。在此次更改之前，  :class:` .Column`  对象在  :paramref:`_orm.Mapper.properties`  中的分配顺序将始终在映射的   :class:` .Table`  的映射列之前进行映射，导致属性在映射类上的分配顺序与他们在从映射类生成的 ORM SELECT 语句中出现的列的顺序不一致的情况。

        更改最突出的方面发生在declarative向   :class:`.Mapper`  中声明列的分配，特别是在列DDL名称与映射属性名称明确不同时的情况，以及在使用类似   :func:` _orm.deferred`  等构造时。在   :class:`.Mapper`  中分配属性时，这种新行为将确保映射到可映射   :class:` .Table`  中的列的顺序与在映射类上映射属性的顺序相同，并分配给   :class:`.Mapper`  本身，以及在 ORM 语句中呈现的顺序相同，而不管   :class:` _schema.Column`  在   :class:`.Mapper`  中是怎样配置的。

    .. change::
        :tags: bug, orm, declarative
        :tickets: 8718

        改进了类   :class:`.DeclarativeBase` ，使其与其他混合元类，如   :class:` .MappedAsDataclass`  结合使用时，类的顺序可以是任意的。

    .. change::
        :tags: usecase, declarative, orm
        :tickets: 8665

        支持映射的类也是 `Generic` 的子类。

    .. change::
        :tags: bug, sql
        :tickets: 8849

        重写了 "numeric" 参数样式的方法，现在完全支持，包括由 "展开IN" 和 "insertmanyvalues" 触发的特殊处理。源 SQL 结构中的参数名称也可以重复，其只占用一个要素，具有难以置信的功能。引入了一个名为 `numeric_dollar` 的其他数字参数样式，它是异步 pg 那样使用美元符号指示数字的位置。asyncpg 方言现在直接使用 'numeric_dollar' 样式。

        'numeric' 和 'numeric_dollar' 参数样式假定目标后端能够按任意顺序接收数字参数，并将给定的参数值与语句匹配，基于匹配它们的位置（从1开始）到数字指示器。这是 "numeric" 参数样式的正常行为，尽管观察到 SQLite DBAPI 实现了一个未使用的 "numeric" 样式，不遵守参数顺序。

    .. change::
        :tags: usecase, postgresql
        :tickets: 8765

        补充了  :ticket:`8690` ，增加了针对 PG 特定范围对象的新比较方法，如  :meth:` _postgresql.Range.adjacent_to` ，  :meth:`_postgresql.Range.difference`  ，  :meth:` _postgresql.Range.union`   等，这使它们与底层的  :attr:`_postgresql.AbstractRange.comparator_factory`  实现的标准运算符相媲美。 此外，类的 ` `__bool __（）`` 方法已被更正，以与普通 Python 容器行为以及其他流行的 PostgreSQL 驱动程序执行的方式一致：它告诉范围实例是否**不**为空，而不是反过来。此变更由 Lele Gaifax 提供。


.. changelog::
    :version: 2.0.0b2
    :released: January 26, 2023
    :released: October 20, 2022

    .. change::
        :tags: bug, orm
        :tickets: 8656

        删除了关于使用 ORM 启用的update/delete 时关于按名称评估列的警告，该警告最初在  :ticket:`4073`  中添加；实际上，此警告掩盖了一个否则的情况，即根据实际的列是什么，可能为 ORM 映射的属性填充错误的 Python 值，因此已删除。

    .. change::
        :tags: bug, typing
        :tickets: 8645

        修复 pylance 严格模式下使用方法来定义 ``__tablename__``、``__mapper_args__`` 或 ``__table_args__`` 时报告的“实例变量覆盖类变量”的问题。

    .. change::
        :tags: mssql, bug
        :tickets: 7211

          :class:`.Sequence`  结构恢复到了 1.4 系列之前的 DDL 行为，即创建没有附加参数的   :class:` .Sequence`  将发出简单的 ``CREATE SEQUENCE`` 指令 **不包含** 任何其他“起始值”的参数。对于大多数后端，这实际上是以前的做法；**然而**，对于 MS SQL Server，此数据库的默认值为 ``-2**63``;为防止在SQL Server上出现非常不实用的此默认值，应提供  :paramref:`~.Sequence.start`  参数。由于长期以来，对于 SQL Server 使用   :class:` .Sequence`  已很少见，而已经标准化为 ``IDENTITY``，因此希望此更改对其影响很小。

        .. seealso::

              :ref:`change_7211` 

    .. change::
        :tags: bug, typing
        :tickets: 8776

        修复了一个在  :paramref:`_orm.relationship.order_by`  中传递返回可迭代列元素的 callable 函数时在类型检查器中报告 "instance variable overrides class variable" 的问题。

我翻译了草稿，下面来一个清晰的翻译:
      
.. changelog::
    :version: 2.0.0b4
    :released: January 26, 2023
    :released: December 5, 2022

    .. change::
        :tags: bug, sql
        :tickets: 8994

        现在针对第三方方言，通过使用可重写的  :attr:`.SQLCompiler.bindname_escape_chars`   字典，系统在绑定的参数名称部分进行特殊字符转义。此更改还添加了点 "." 作为默认转义字符。

    .. change::
        :tags: orm, feature
        :tickets: 8889

        引入了一个名为 "auto" 的  :paramref:`.Mapper.eager_defaults`  参数的新默认值。在一次工作单元的 flush 过程中，如果方言支持 INSERT 的 RETURNING，并且有 insertmanyvalues 可用，将自动获取表格的默认值。如果设置  :paramref:` .Mapper.eager_defaults`  为 "True"，则在表示服务器端 UPDATE 默认值（极不常见）的情况下持续取用； 无法让任何使用  :paramref:`.Mapper.eager_defaults`  的 UPDATE 语句触发对默认值的加载，因为当前并没有用于批处理 RETURNING 的方式。

    .. change::
        :tags: usecase, orm
        :tickets: 8973

        现在无需在 `DeclarativeDataclass Mapped` 类上使用 `__allow_unmapped__` 属性来标记 非 `Mapped[]` 注释中检测到的哪些注释 时。当前不再引发旨在支持遗留 ORM typed 映射的错误消息；更正的错误消息也没有再提供有关如何针对Dataclasses处理的正确模式的信息。这个错误消息将不会再次出现，如果使用了  :meth:`_orm.registry.mapped_as_dataclass`  或   :class:` _orm.MappedAsDataclass` 。

    .. change::
        :tags: bug, orm
        :tickets: 8812

        修复在使用 :paramref:`_orm.Mapper.eager_defaults` 参数时，刷新映射到子查询的映射类（例如直接映射或某些形式的具体表继承）将导致失败的错误，此前的代码可能会触发对默认值的加载。


.. changelog::
    :version: 2.0.0b3
    :released: January 26, 2023
    :released: November 4, 2022

    .. change::
        :tags: bug, orm, declarative
        :tickets: 8759

        为   :class:`.Mapped`  注释添加了 ORM 的支持，支持关联的另一个类名作为第一个参数传入。添加了另一种方法，如果类名没有导入可以使用，即使用关联的 `  :class:` _orm.Mapped` ` 符号的名称作为类名。另外，指定为   :func:`_orm.relationship`  的主参数，传递的类名也将始终优先于注释中给出的名称（因为其它名称可能无法导入）。

    .. change::
        :tags: bug, orm
        :tickets: 8692

        改善了在注释中使用不包括 ``Mapped[]`` 的注释的旧 1.4 映射的支持，通过确保 `__allow_unmapped__` 属性可以使用，始终会使这种旧注释通过 Annotated Declarative 而不激发错误 。此外，改进了检测到此情况时生成的错误消息，并增加了有关如何处理此情况的更多文档。不幸的是，1.4 WARN_SQLALCHEMY_20 迁移警告 **不能** 检测到这种情况。

    .. change::
        :tags: usecase, postgresql
        :tickets: 8690

        优化了在  :ref:`change_7156` 中描述的新 PostgreSQL   :class:` .Range`  对象的处理方式，以适应特定于驱动程序的范围和多范围对象，以更好地适应遗留代码和从原始 SQL 结果集传回新范围或多范围表达式的情况。

    .. change::
        :tags: usecase, engine
        :tickets: 8717

        添加了新参数  :paramref:`.PoolEvents.reset.reset_state`  到  :meth:` .PoolEvents.reset`  事件，其中包含有关如何重置连接的各种状态信息，以允许具有完整上下文的自定义重置计划运行。在此更改中还包括修复，在 1.4 版本中重新启用  :meth:`.PoolEvents.reset`  事件，使其在所有情况下都能运行，包括   :class:` .Connection`  已经 "重置" 连接。这两个变化共同允许使用  :meth:`.PoolEvents.reset`  事件来实现自定义重置计划，而不是  :meth:` .PoolEvents.checkin`  事件。

    .. change::
        :tags: bug, orm, declarative
        :tickets: 8705

        修改了   :class:`.Mapper`  的基本配置行为，此前当  :paramref:` _orm.Mapper.properties`  字典中明确存在   :class:`_schema.Column`  对象时，无论是直接还是封装在映射器属性对象中，它们都将在映射到可映射   :class:` .Table` （或其他可选择性）本身（前提是它们实际上属于该表的列列表）中，保持所映射的   :class:`.Table`  列的相同顺序，从而保持分配到映射类的属性的顺序与映射类上的封装以及在 ORM SELECT 语句中呈现的属性的顺序一致。在此更改之前，  :class:` .Column`  对象在  :paramref:`_orm.Mapper.properties`  中的分配顺序将始终在映射的   :class:` .Table`  的映射列之前进行映射，导致属性在映射类上的分配顺序与他们在从映射类生成的 ORM SELECT 语句中出现的列的顺序不一致的情况。

        更改最突出的方面发生在declarative向   :class:`.Mapper`  中声明列的分配，特别是在列DDL名称与映射属性名称明确不同时的情况，以及在使用类似   :func:` _orm.deferred`  等构造时。在   :class:`.Mapper`  中分配属性时，这种新行为将确保映射到可映射   :class:` .Table`  中的列的顺序与在映射类上映射属性的顺序相同，并分配给   :class:`.Mapper`  本身，以及在 ORM 语句中呈现的顺序相同，而不管   :class:` _schema.Column`  在   :class:`.Mapper`  中是怎样配置的。


.. changelog::
    :version: 2.0.0b2
    :released: January 26, 2023
    :released: October 20, 2022

    .. change::
        :tags: bug, orm
        :tickets: 8656

        移除了关于使用 ORM 启用的 update/delete 时关于按名称评估列的警告，该警告最初在  :ticket:`4073`  中添加。该警告掩盖了一个否则的情况，即根据实际的列是什么，可能为 ORM 映射的属性填充错误的 Python 值，因此已删除。

    .. change::
        :tags: bug, typing
        :tickets: 8645

        修复 pylance 严格模式下使用方法来定义 ``__tablename__``、``__mapper_args__`` 或 ``__table_args__`` 时报告的“实例变量覆盖类变量”的问题。

    .. change::
        :tags: mssql, bug
        :tickets: 7211

          :class:`.Sequence`  结构恢复到了 1.4 系列之前的 DDL 行为，即创建没有附加参数的   :class:` .Sequence`  将发出简单的 ``CREATE SEQUENCE`` 指令 **不包含** 任何其他“起始值”的参数。对于大多数后端，这实际上是以前的做法；**然而**，对于 MS SQL Server，此数据库的默认值为 ``-2**63``;为防止在SQL Server上出现非常不实用的此默认值，应提供  :paramref:`~.Sequence.start`  参数。由于长期以来，对于 SQL Server 使用   :class:` .Sequence`  已很少见，而已经标准化为 ``IDENTITY``，因此希望此更改对其影响很小。

        .. seealso::

              :ref:`change_7211` 

    .. change::
        :tags: bug, typing
        :tickets: 8776

        修复在  :paramref:`_orm.relationship.order_by`  中传递返回可迭代列元素的 callable 函数时在类型检查器中报告 "instance variable overrides class variable" 的问题。.. _2.0.0b1:

版本 2.0.0b1
=============

发布日期：2023年1月26日

.. change::
    :tags: bug, orm, declarative
    :tickets: 8668

    修复了新的 ORM 强类型映射中出现的错误。可以在多对一关系的类型标注中使用 ``Optional[MyClass]`` 或类似形式，例如 ``MyClass | None``，这将修复错误。相关文档也已添加到关系配置文档中。

.. change::
    :tags: bug, typing
    :tickets: 8644

    修复 pylance 严格模式下   :func:`_orm.mapped_column`  构造函数报告类型为“部分未知”的问题。

.. change::
    :tags: bug, regression, sql
    :tickets: 8639

    修复了新的“insertmanyvalues”功能中的错误。使用了包含   :func:`_sql.bindparam`  的子查询的 INSERT 将无法正确呈现在“insertmanyvalues”格式中。这主要影响 psycopg2 库，因为它无条件地使用“insertmanyvalues”。

.. change::
    :tags: bug, orm, declarative
    :tickets: 8688

    修复使用新的数据类映射功能时的问题。在处理覆盖   :func:`_orm.mapped_column`  声明的混合类时，可能会导致传递给 dataclasses API 的参数顺序出现问题，导致初始化程序中出现问题。

.. change::
    :tags: bug, sql
    :tickets: 7888

    在使用  :meth:`_sql.Select.select_from`  方法时，建立在   :func:` _sql.select`  构造函数之上的 FROM 从句现在将首先呈现在所呈现 SELECT 的 FROM 从句中，这有助于维护在传递给  :meth:`_sql.Select.select_from`  方法时保持的子句顺序而不受它们在查询的其他部分中也被提到的影响。对所有支持的数据库，这种改进都非常有用，以允许针对特定的 FROM 从句顺序生成期望的查询计划，并完全控制 FROM 从句的顺序。

.. change::
    :tags: usecase, sql
    :tickets: 7998

    针对   :class:`_dml.Insert`  构造函数的编译机制进行了更改，以便即使存在于参数集中或在  :meth:` _dml.Insert.values`  方法中作为普通绑定值提供，只要使用了“自增长主键”列值，即使在传递显式 NULL 时，也将通过 ``cursor.lastrowid`` 或 RETURNING 获取它，用于针对已知生成自增长值的特定后端的单行插入语句。这还原了 1.3 系列中的行为，对于分别使用参数集和  :meth:`_dml.Insert.values`  的情况都是如此。在 1.4 中，参数集行为无意间改变成了不再使用此操作，但  :meth:` _dml.Insert.values`  方法在 1.4.21 之前仍可以获取自增长值，此时  :ticket:`6770`  再次意外更改了行为。

    该行为的定义是“可以工作”，以适应像 SQLite、MySQL 和 MariaDB 等数据库，它们将忽略显式 NULL 主键值，尽管显式 NULL 主键值将被提供，但仍会调用自增长生成器。

    .. seealso::

          :ref:`external_toplevel` 

    .. change::
        :tags: bug, orm
        :tickets: 7463

        修复了性能回归的问题，该问题至少在 1.3 版本中出现（在 1.0 之后某个时间点之后出现），其中从连接的子类中加载 deferred 列（那些显式映射为   :func:`_orm.defer`  的列，而不是过期的非 deferred 列）将不使用“优化”查询，该查询仅查询包含未加载列的直接表，而是运行全 ORM 查询，该查询将为所有基表发出 JOIN，这在仅从子类加载列时是不必要的。

    .. change::
        :tags: bug, sql
        :tickets: 7791

        当使用非原生枚举类型的   :class:`_sqltypes.Enum`  数据类型时，  :paramref:` .Enum.length`   参数（用于设置 ``VARCHAR`` 列的长度）现已无条件使用，包括在标准使用了  :paramref:`.Enum.native_enum`  参数设置为 ` `True`` 的后端，后端继续使用 ``VARCHAR`` 数据类型。此前，该参数在这种情况下将被错误忽略。现在已删除此情况下曾出现的警告。

    .. change::
        :tags: feature, orm
        :tickets: 6986

        多数与   :class:`_orm.Load`  对象相关的内部工作和相关的加载策略模式已经被大部分重写，大部分重写是为了利用这样的事实：现在只支持属性绑定路径，而不是字符串。该重写希望使在加载策略系统中针对新用例和微妙问题更加简单明了。

    .. change::
        :tags: usecase, orm

        为   :func:`_orm.load_only`  loader 选项添加了  :paramref:` _orm.load_only.raiseload` 
        参数，以便加载项可以通过 "raise" 行为而不是懒惰加载来获取未加载属性。先前无法直接使用   :func:`_orm.load_only`  选项实现此目的。

    .. change::
        :tags: change, engine
        :tickets: 7122

        有关引擎和方言的一些 API 更改：

        *  :meth:`.Dialect.set_isolation_level` 、  :meth:` .Dialect.get_isolation_level`  
          和 :meth:
          换句话说，不带有由 SQLAlchemy 传递的任何种类的封装。

        *   :class:`.Connection`  和   :class:` .Engine`  类不再共享共同的超类 "Connectable"，已将其删除。

        * 添加了一个新的接口类   :class:`.PoolProxiedConnection` ，这是公共接口，用于熟悉的   :class:` ._ConnectionFairy`  类，尽管它是一个私有类。

    .. change::
        :tags: feature, sql
        :tickets: 3482

        添加了期待已久的不区分大小写的字符串操作符  :meth:`_sql.ColumnOperators.icontains` 、  :meth:` _sql.ColumnOperators.istartswith`  、  :meth:`_sql.ColumnOperators.iendswith`  ，它们生成不区分大小写的 LIKE 组成部分（在 PostgreSQL 中使用 ILIKE，在所有其他后端中使用 LOWER() 函数），以补充现有的 LIKE 组成操作符，如  :meth:` _sql.ColumnOperators.contains` 、  :meth:`_sql.ColumnOperators.startswith`   等。非常感谢 Matias Martinez Rebori 在实现这些新方法方面的细致而详尽的努力。

    .. change::
        :tags: usecase, postgresql
        :tickets: 8138

        为   :class:`_sqltypes.ARRAY`  和   :class:` _postgresql.ARRAY`  数据类型添加了文本类型化。使用通用字符串嵌套方括号方式呈现，例如 ``[1, 2, 3]``， Postgres 特定的呈现方式使用 ARRAY 文本，例如 ``ARRAY[1, 2, 3]``。还考虑了多个维度和引号。

    .. change::
        :tags: bug, orm
        :tickets: 8166

        对“deferred”/"load_only" 策略选项进行了改进，在同一查询中从两条不同的逻辑路径加载某个对象时，已配置成应填充的属性将在所有情况下都填充，即使该对象的其他加载路径没有设置此选项。以前，它是基于哪个“路径”先处理对象的随机性。

    .. change::
        :tags: feature, orm, sql
        :tickets: 6047

        对所有支持 RETURNING 的分数执行方言，包括所有 PostgreSQL 驱动程序、SQLite、MariaDB、MS SQL Server，添加了新功能“insertmanyvalues”。这是一种将 ORM 插入语句分批成一个更加高效的 SQL 结构的一般方法，而仍能够使用 RETURNING 获取新生成的主键和 SQL 默认值。

        该功能现在适用于许多支持 RETURNING 和多个 INSERT 中的 VALUES 构造的分数，包括所有 PostgreSQL 驱动程序、SQLite、MariaDB 和 MS SQL Server。此外，Oracle 方言也使用本机 cx_Oracle 或 OracleDB 特性获得了相同的功能。

    .. change::
        :tags: bug, engine
        :tickets: 8523

          :class:`_pool.QueuePool`  现在在 ` `pool_size=0`` 时忽略 ``max_overflow``，以便在所有情况下都是无限制的。

    .. change::
        :tags: bug, sql
        :tickets: 7909

        对 Python 整数进行就地类型检测，就像使用表达式 ``literal(25)`` 一样，现在还适配了 Python 大整数，其中决定的数据类型将是   :class:`.BigInteger`  而不是   :class:` .Integer` 。这适应于诸如 asyncpg 的方言，该方言同时将隐式类型信息发送到驱动程序，并对数字刻度敏感。

    .. change::
        :tags: postgresql, mssql, change
        :tickets: 7225

          :class:`_types.UUID`  中  :paramref:` _types.UUID.as_uuid`  参数的默认值现在为 ``True``，这表明该参数默认情况下接受 Python ``UUID`` 对象。此外，SQL Server   :class:`_mssql.UNIQUEIDENTIFIER`  数据类型已转换为可接受 UUID 的数据类型。对于使用字符串值的   :class:` _mssql.UNIQUEIDENTIFIER`  的遗留代码，请将  :paramref:`_mssql.UNIQUEIDENTIFIER.as_uuid`  参数设置为 ` `False``。

    .. change::
        :tags: bug, orm
        :tickets: 8344

        修复了在 ORM 启用的 UPDATE 表达式中使用连接继承子类创建语句时，仅更新本地表列的情况下，使用“fetch”同步策略会导致致命错误，这对于使用会使用 RETURNING 进行同步的数据库进行操作是不会呈现正确的，还调整了 UPDATE FROM 和 DELETE FROM 语句中使用的 RETURNING 策略。

    .. change::
        :tags: usecase, mariadb
        :tickets: 8344

        为 ORM 启用的 DELETE 语句添加了名为“is_delete_using=True”的新执行选项，该选项在与“fetch”同步策略一起使用 ORM 启用的 DELETE 语句时表示 DELETE 语句预计将使用多个表，在 MariaDB 上是 DELETE..USING 语法。此后，即使已知不支持“DELETE..USING..RETURNING”语法但支持“DELETE..USING”的数据库，请在 ORM 中不使用返回（在 SQLAlchemy 2.0 中针对 MariaDB 进行修复，针对  :ticket:`7011` ）。理由是 ORM 启用的 DELETE 不能准确地知道 DELETE 语句是针对多个表还是针对单个表，直到编译完成，而编译已被缓存，但需要知道 DELETE 操作期望将删除的行中比较特殊的行。相对于为这个相对罕见的 SQL 模式主动检查所有 DELETE 语句而产生全局性能作用，现在通过编译阶段中引发的新异常消息并使用 ` `is_delete_using=True`` 执行选项来请求一个。如果此异常消息未在编译步骤中提供，在任何情况下都不会此情况。透过执行选项，ORM 知道该在前面运行 SELECT。ORM 启用的 UPDATE 同样实现了类似的选项，但是当前还没有需要此选项的后端。

    .. change::
        :tags: bug, orm, asyncio
        :tickets: 7703

        从   :class:`_asyncio.AsyncSession.begin`  和   :class:` _asyncio.AsyncSession.begin_nested`  中删除了未使用的 ``**kw`` 参数。这些关键字参数由于错误地添加到 API 中而没有使用。

    .. change::
        :tags: feature, sql
        :tickets: 8285

        向所有   :class:`.FromClause`  对象的  :attr:` .FromClause.c`  集合添加了新语法，该语法允许通过使用键元组传递给 ``__getitem__()`` 的方式，并通过   :func:`_sql.select`  构造函数直接处理结果，从而允许使用语法 ` `select(table.c['a', 'b', 'c'])``。返回的子集本身是   :class:`.ColumnCollection` ，也可以直接被   :func:` _sql.select`  和类似函数使用。

        .. seealso::

              :ref:`tutorial_selecting_columns` 

    .. change::
        :tags: general, changed
        :tickets: 7257

        迁移代码库以删除所有在 2.0 中已经指明已弃用以待删除的预 2.0 行为和架构，包括但不限于：

          * 删除了所有 Python 2 代码，最低版本现在是 Python 3.7。

          * 现在   :class:`_engine.Engine`  和   :class:` _engine.Connection`  使用了新的-2.0 样式，包括 "autobegin"，已删除库级别自动提交，已删除子事务和“分支”连接

          * Result 对象使用 2.0 样式行为，   :class:`_result.Row`  完全是一个命名元组，没有 "mapping" 行为，对于具有“映射”行为的需要使用   :class:` _result.RowMapping` 

          * 所有 Unicode 编码/解码架构已从 SQLAlchemy 中删除。所有现代的 DBAPI 实现都支持 Unicode，这归功于 Python 3，因此将删除 DBAPI ``cursor.description`` 等中的字节串相关机制。

          *   :class:`.MetaData` 、  :class:` .Table`  及其所有 DDL/DML/DQL 元素中的 .bind 属性和参数在先前可以将其引用为“绑定引擎”。现在，在所有情况下都是使用连接 URLs 创建引擎和连接实例，因此已删除。

          * 单独的 ``sqlalchemy.orm.mapper()`` 函数已删除；所有经典映射都应通过   :class:`_orm.registry` 
            的  :meth:`_orm.registry.map_imperatively`  方法完成。

          *  :meth:`_orm.Query.join`  方法不再接受字符串作为关系名称；现在正式标准化了使用“Class.attrname”作为连接目标的长时间文档化方法。

          *  :meth:`_orm.Query.join`  不再接受“aliased”和“from_joinpoint”参数

          *  :meth:`_orm.Query.join`  不再在一个方法调用中接受多个连接目标的链。

          * ``Query.from_self()``、``Query.select_entity_from()`` 和 ``Query.with_polymorphic()`` 已删除。

          *  :paramref:`_orm.relationship.cascade_backrefs`  参数现在必须保持其默认值
            “False”；“save-update”级联不再沿着反向引用级联。

          *  :paramref:`_orm.Session.future`  参数现在必须始终设置为 ` `True``。现在始终启用 2.0 样式的   :class:`_orm.Session` 
            事务模式。

          * 加载选项不再接受属性名称字符串。加载选项的通常文档化方法是针对加载选项目标使用“Class.attrname”。

          * 经典“_sql.select”已删除，包括
            ``select([cols])``，“whereclause”和 ``some_table.select()`` 的关键字参数。

          *   :class:`_sql.Select`  的遗留“原地变异器”方法，例如 ` `append_whereclause()``
            和 ``append_order_by()`` 等，已删除。

          * 删除了非常古老的“dbapi_proxy”模块，在非常早期的 SQLAlchemy 版本中使用它提供了一个透明连接池来管理原始的 DBAPI 连接。

    .. change::
        :tags: feature, orm
        :tickets: 8375

        添加了新参数  :paramref:`_orm.AttributeEvents.include_key` ，用于包括被视为操作的字典或列表键（例如“obj[key] = value”或“del obj[key]”）的属性，使用新的关键字参数“key”或“keys”，具体取决于事件，例如  :paramref:` _orm.AttributeEvents.append.key` 、  :paramref:`_orm.AttributeEvents.bulk_replace.keys`  。这允许事件处理程序考虑传递给操作的键，并且对于工作使用   :class:` _orm.MappedCollection`  的字典操作来说非常重要。

    .. change::
        :tags: postgresql, usecase
        :tickets: 7156, 8540

        为 PostgreSQL 添加多范围数据类型，这些类型在 PostgreSQL 14 中引入。现在，支持 PostgreSQL 范围和多范围的支持已经推广到了 psycopg3、psycopg2 和 asyncpg 后关端，可通过一种与先前使用的 psycopg2 对象构造函数兼容的后端不可知的   :class:`_postgresql.Range`  数据对象进行。有关使用模式的新文档。

        此外，范围类型处理已得到增强，使得它自动呈现类型转换，以便于一般而言，对于未提供数据库任何上下文的语句执行原位往返时，并不需要明确地使用   :func:`_sql.cast`  构造函数来获取所需的类型（在  :ticket:` 8540`  中讨论）。

        非常感谢 @zeeeeeb 的拉请求，实现并测试了新的数据类型和 psycopg 支持。

        .. seealso::

              :ref:`change_7156` 

              :ref:`postgresql_ranges` 

    .. change::
        :tags: usecase, oracle
        :tickets: 8221

        现在，Oracle 将默认使用 FETCH FIRST N ROWS / OFFSET 语法以支持 Oracle 12c 和以上版本的限制/偏移量支持。当直接使用  :meth:`_sql.Select.fetch`  时，此语法已经可用，现在在  :meth:` _sql.Select.limit`  和  :meth:`_sql.Select.offset`  上也是如此。

    .. change::
        :tags: bug, engine
        :tickets: 8567

        为了提高安全性，当调用 ``str(url)`` 时，  :class:`_url.URL`  对象现在默认使用密码混淆。要使用明文密码字符串化 URL，可以使用  :meth:` _url.URL.render_as_string` ，并传递  :paramref:`_url.URL.render_as_string.hide_password`  参数为 ` `False``。感谢我们的贡献者提供了此拉请求。

        .. seealso::

              :ref:`change_8567` 

    .. change::
        :tags: change, orm

        为了更好地适应显式类型，通常情况下，在内部构造，并且有时会在消息中可见的某些 ORM 构造的名称已更改为更简洁的名称，这些名称也匹配（使用不同大小写）它们的构造函数名称，目前在所有情况下都保留旧名称的别名：

        *   :class:`_orm.Relationship`  现在作为主要名称，  :class:` _orm.RelationshipProperty`  变为一个别名，两者都是使用   :func:`_orm.relationship`  函数构造的。

        *   :class:`_orm.Synonym`  现在作为主要名称，  :class:` _orm.SynonymProperty`  变为一个别名，两者都是使用   :func:`_orm.synonym`  函数构建的。

        *   :class:`_orm.Composite`  现在作为主要名称，  :class:` _orm.CompositeProperty`  变为一个别名，两者都是使用   :func:`_orm.composite`  函数构建的。

    .. change::
        :tags: orm, change
        :tickets: 8608

        为了更加符合显式编程概念   :class:`_orm.Mapped` ，一些字典定向的集合名称，   :func:` _orm.attribute_mapped_collection` 、  :func:`_orm.column_mapped_collection`  和   :class:` _orm.MappedCollection`  已更改为   :func:`_orm.attribute_keyed_dict` 、  :func:` _orm.column_keyed_dict`  和   :class:`_orm.KeyFuncDict` ，使用“dict”短语，以最小化任何与术语“映射”有关的混淆。旧名称将无限期保留，没有删除计划。

    .. change::
        :tags: bug, sql
        :tickets: 7354

        为所有“Create”/“Drop”构造函数添加了``if_exists``和``if_not_exists``参数，包括   :class:`.CreateSequence` 、  :class:` .DropSequence` 、  :class:`.CreateIndex` 、  :class:` .DropIndex`  等，允许在 DDL 中呈现通用“IF EXISTS” /“IF NOT EXISTS”短语。来自 Jesse Bakker 的拉取请求。

    .. change::
        :tags: engine, usecase
        :tickets: 6342

        加强了基础方言的  :paramref:`_sa.create_engine.isolation_level`  参数，因此不再依赖于各个方言的存在。此参数在创建所有新的数据库连接在由连接池创建时立即设置“隔离级别”设置，此后该值保持设置而不被重置。

         :paramref:`_sa.create_engine.isolation_level`  参数在功能上与使用  :meth:` _engine.Engine.execution_options`  中的  :paramref:`_engine.Engine.execution_options.isolation_level`  参数为引擎级别的设置基本上是等效的。不同之处在于前者在创建连接时仅分配隔离级别一次，而后者在每次连接检出时设置和重置给定的级别。.. _change_7433:

7433 - orm: Improved error messages on stolen state
----------------------------------------------------

The   :class:`_orm.Session`  (and by extension   :class:` .AsyncSession` ) now has
new state-tracking functionality that will proactively trap any unexpected
state changes which occur as a particular transactional method proceeds.
This is to allow situations where the   :class:`_orm.Session`  is being used in
a thread-unsafe manner, where event hooks or similar may be calling
unexpected methods within operations, as well as potentially under other
concurrency situations such as asyncio or gevent to raise an informative
message when the illegal access first occurs, rather than passing silently
leading to secondary failures due to the   :class:`_orm.Session`  being in an
invalid state. 

Related tickets:  :ticket:`7433` 

.. seealso::

      :ref:`session_threading` 

.. versionadded:: 1.4.0b1

.. _change_6842:

6842 - postgresql: Add psycopg2 dialect
----------------------------------------

Added support for ``psycopg2`` dialect supporting both sync and async
execution. This dialect is available under the ``postgresql+psycopg2`` name
for both the   :func:`_sa.create_engine`  and   :func:` _asyncio.create_async_engine` 
engine-creation functions.

Related tickets:  :ticket:`6842` 

.. seealso::

      :ref:`dialects_postgresql_psycopg2` 

.. versionadded:: 1.4.0b1

.. _change_6195:

6195 - sqlite: Support RETURNING on inserts
--------------------------------------------

Added RETURNING support for the SQLite dialect. SQLite supports RETURNING
since version 3.35.

Related tickets:  :ticket:`6195` 

.. versionadded:: 1.4.0b1

.. _change_7011:

7011 - mariadb: Support insert and delete returning clauses
------------------------------------------------------------

Added INSERT..RETURNING and DELETE..RETURNING support for the MariaDB
dialect.  UPDATE..RETURNING is not yet supported by MariaDB.  MariaDB
supports INSERT..RETURNING as of 10.5.0 and DELETE..RETURNING as of
10.0.5.

Related tickets:  :ticket:`7011` 

.. seealso::

      :ref:`dialects_mariadb` 

.. versionadded:: 1.4.0b1

.. _change_composite_autoreload:

Allowed automatic resolution of values for composite mapping with dataclass
----------------------------------------------------------------------------

The   :func:`_orm.composite`  mapping construct now supports automatic resolution of
values when used with a Python ``dataclass``; the ``__composite_values__()`` method
no longer needs to be implemented as this method is derived from inspection of the
dataclass. Additionally, classes mapped by   :class:`_orm.composite`  now support
ordering comparison operations, e.g. ``<``, ``>=``, etc.

See the new documentation at   :ref:`mapper_composite`  for examples.

.. seealso::

      :ref:`mapping_composite` 

.. versionadded:: 1.4.0b1

.. _change_7161:

7161 - engine: Use consistent behavior for view/table interplay
---------------------------------------------------------------

The  :meth:`_engine.Inspector.has_table`  method will now consistently check for
views of the given name as well as tables. Previously this behavior was dialect
dependent, with PostgreSQL, MySQL/MariaDB and SQLite supporting it, and Oracle and
SQL Server not supporting it. Third party dialects should also seek to ensure their
  :meth:`_engine.Inspector.has_table`   method searches for views as well as tables
for the given name.

Related tickets:  :ticket:`7161` 

.. versionchanged:: 1.4.0b1

.. _change_5648:

5648 - engine: Dialect event handle_error moved to DialectEvents, now participates in connection pool 'pre ping'
---------------------------------------------------------------------------------------------------------------

The  :meth:`.DialectEvents.handle_error`  event is now moved to the
  :class:`.DialectEvents`  suite from the   :class:` .EngineEvents`  suite, and now
participates in the connection pool "pre ping" event for those dialects that make
use of disconnect codes in order to detect if the database is live. This allows
end-user code to alter the state of "pre ping". Note that this does not include
dialects which contain a native "ping" method such as that of psycopg2 or most
MySQL dialects. 

.. versionchanged:: 1.4.0b1

.. _change_7212:

7212 - sql: Newly available core type for UUID
-----------------------------------------------

Added new backend-agnostic   :class:`_types.Uuid`  datatype generalized from
the PostgreSQL dialects to now be a core type, as well as migrated
  :class:`_types.UUID`  from the PostgreSQL dialect. The SQL Server
  :class:`_mssql.UNIQUEIDENTIFIER`  datatype also becomes a UUID-handling datatype.
Thanks to Trevor Gross for the help on this.

Related tickets:  :ticket:`7212` 

.. seealso::

      :ref:`types_core_uuid` 

.. versionadded:: 1.4.0b1

.. _change_8126:

8126 - orm: Very experimental feature added to selectinload and immediateload
-----------------------------------------------------------------------------

Added very experimental feature to the   :func:`_orm.selectinload`  and
  :func:`_orm.immediateload`  loader options called
  :paramref:`_orm.selectinload.recursion_depth`   /
  :paramref:`_orm.immediateload.recursion_depth`   , which allows a single loader option
to automatically recurse into self-referential relationships. Is set to an integer
indicating depth, and may also be set to -1 to indicate to continue loading until
no more levels deep are found. Major internal changes to   :func:`_orm.selectinload` 
and   :func:`_orm.immediateload`  allow this feature to work while continuing to make
correct use of the compilation cache, as well as not using arbitrary recursion,
so any level of depth is supported (though would emit that many queries).  This may
be useful for self-referential structures that must be loaded fully eagerly, such
as when using asyncio.

A warning is also emitted when loader options are connected together with
arbitrary lengths (that is, without using the new ``recursion_depth`` option) when
excessive recursion depth is detected in related object loading. This operation
continues to use huge amounts of memory and performs extremely poorly; the cache is
disabled when this condition is detected to protect the cache from being flooded
with arbitrary statements.

Related tickets:  :ticket:`8126` 

.. versionadded:: 1.4.0b1

.. _change_8403:

8403 - orm: New parameter to limit scope of attributes on subclasses declared by AbstractConcreteBase
-------------------------------------------------------------------------------------------------------

Added new parameter  :paramref:`.AbstractConcreteBase.strict_attrs`  to the
  :class:`.AbstractConcreteBase`  declarative mixin class. The effect of this parameter
is that the scope of attributes on subclasses is correctly limited to the subclass
in which each attribute is declared, rather than the previous behavior where all
attributes of the entire hierarchy are applied to the base "abstract" class. This
produces a cleaner, more correct mapping where subclasses no longer have non-useful
attributes on them which are only relevant to sibling classes. The default for this
parameter is False, which leaves the previous behavior unchanged; this is to support
existing code that makes explicit use of these attributes in queries. To migrate to
the newer approach, apply explicit attributes to the abstract base class as needed.

.. versionadded:: 1.4.0b1

.. _change_8503:

8503 - mysql, mariadb: Support group by rollup syntax
------------------------------------------------------

The ``ROLLUP`` function will now correctly render ``WITH ROLLUP`` on MySql and
MariaDB, allowing the use of group by rollup with these backends.

Related tickets:  :ticket:`8503` 

.. versionadded:: 1.4.0b1

.. _change_6928:

6928 - orm: Add optional flag to disable implicit transactions
----------------------------------------------------------------

Added new parameter  :paramref:`_orm.Session.autobegin` , which when set to ` `False``
will prevent the   :class:`_orm.Session`  from beginning a transaction implicitly. The
  :meth:`_orm.Session.begin`   method must be called explicitly first in order to
proceed with operations, otherwise an error is raised whenever any operation would
otherwise have begun automatically. This option can be used to create a "safe"
  :class:`_orm.Session`  that won't implicitly start new transactions.

As part of this change, also added a new status variable
  :class:`_orm.SessionTransaction.origin`  which may be useful for event handling code to
be aware of the origin of a particular   :class:`_orm.SessionTransaction` .

.. versionadded:: 1.4.0b1

.. _change_7256:

7256 - platform: Replace C extensions with Cython
-------------------------------------------------

The SQLAlchemy C extensions have been replaced with all new implementations written
in Cython.  Like the C extensions before, pre-built wheel files for a wide range of
platforms are available on pypi so that building is not an issue for common
platforms.  For custom builds, ``python setup.py build_ext`` works as before, needing
only the additional Cython install.  ``pyproject.toml`` is also part of the source
now which will establish the proper build dependencies when using pip.

Related tickets:  :ticket:`7256` 

.. versionadded:: 1.4.0b1

.. _change_7311:

7311 - deprecations: Remove implicit setup.py build from source distribution
-----------------------------------------------------------------------------

SQLAlchemy's source build and installation now includes a ``pyproject.toml`` file
for full  :pep:`517`  support.

Related tickets:  :ticket:`7311` 

.. versionadded:: 1.4.0b1

.. _change_7631:

7631 - schema: Improved support for conditional DDL
---------------------------------------------------

Expanded on the "conditional DDL" system implemented by the
  :class:`_schema.ExecutableDDLElement`  class (renamed from   :class:` _schema.DDLElement` )
to be directly available on   :class:`_schema.SchemaItem`  constructs such as
  :class:`_schema.Index` ,   :class:` _schema.ForeignKeyConstraint` , etc. such that the
conditional logic for generating these elements is included within the default DDL
emitting process. This system can also be accommodated by a future release of
Alembic to support conditional DDL elements within all schema-management systems.

Related tickets:  :ticket:`7631` 

.. versionadded:: 1.4.0b1

.. _change_4379:

4379 - oracle: Reflect materialized views as views, add inspection method for materialized views
------------------------------------------------------------------------------------------------

Materialized views on oracle are now reflected as views. On previous versions of
SQLAlchemy the views were returned among the table names, not among the view names.
As a side effect of this change they are not reflected by default by
  :meth:`_sql.MetaData.reflect`  , unless ` `views=True`` is set. To get a list of
materialized views, use the new inspection method
  :meth:`.Inspector.get_materialized_view_names`  .

Related tickets:  :ticket:`4379` 

.. versionchanged:: 1.4.0b1

.. _change_7299:

7299 - sqlite: Stop warning about non-native Decimal handling
-------------------------------------------------------------

Removed the warning that emits from the   :class:`_types.Numeric`  type about DBAPIs not
supporting Decimal values natively. This warning was oriented towards SQLite, which
does not have any real way without additional extensions or workarounds of handling
precision numeric values more than 15 significant digits as it only uses floating
point math to represent numbers. As this is a known and documented limitation in
SQLite itself, and not a quirk of the pysqlite driver, there's no need for SQLAlchemy
to warn for this. The change does not otherwise modify how precision numerics are
handled. Values can continue to be handled as ``Decimal()`` or ``float()`` as
configured with the   :class:`_types.Numeric` ,   :class:` _types.Float`  , and related
datatypes, just without the ability to maintain precision beyond 15 significant
digits when using SQLite, unless alternate representations such as strings are used.

Related tickets:  :ticket:`7299` 

.. versionchanged:: 1.4.0b1

.. _change_8177:

8177 - mssql: Added use_setinputsizes=True by default for non-unicode string compatibility
------------------------------------------------------------------------------------------

The ``use_setinputsizes`` parameter for the ``mssql+pyodbc`` dialect now defaults to
``True``; this is so that non-unicode string comparisons are bound by pyodbc to
``pyodbc.SQL_VARCHAR`` rather than ``pyodbc.SQL_WVARCHAR``, allowing indexes against
VARCHAR columns to take effect. In order for the ``fast_executemany=True`` parameter
to continue functioning, the ``use_setinputsizes`` mode now skips the
``cursor.setinputsizes()`` call specifically when ``fast_executemany`` is True and
the specific method in use is ``cursor.executemany()``, which doesn't support
setinputsizes. The change also adds appropriate pyodbc DBAPI typing to values that are
typed as   :class:`_types.Unicode`  or   :class:` _types.UnicodeText` , as well as altered
the base   :class:`_types.JSON`  datatype to consider JSON string values as
  :class:`_types.Unicode`  rather than   :class:` _types.String` .

Related tickets:  :ticket:`8177` 

.. versionchanged:: 1.4.0b1

.. _change_7490:

7490 - sqlite: Default to using QueuePool, which holds onto connections when using file-based databases for performance
------------------------------------------------------------------------------------------------------------------------

The SQLite dialect now defaults to   :class:`_pool.QueuePool`  when a file based database
is used. This is set along with setting the ``check_same_thread`` parameter to
``False``. It has been observed that the previous approach of defaulting to
  :class:`_pool.NullPool` , which does not hold onto database connections after they are
released, did in fact have a measurable negative performance impact. As always, the
pool class is customizable via the  :paramref:`_sa.create_engine.poolclass` 
parameter.

Related tickets:  :ticket:`7490` 

.. versionchanged:: 1.4.0b1

.. _change_8141:

8141 - schema: Add IF EXISTS clause to drop constraints
-------------------------------------------------------

Added parameter  :paramref:`_ddl.DropConstraint.if_exists`  to the   :class:` _ddl.DropConstraint` 
construct which result in "IF EXISTS" DDL being added to the DROP statement. This phrase
is not accepted by all databases and the operation will fail on a database that does
not support it as there is no similarly compatible fallback within the scope of a
single DDL statement.

Related tickets:  :ticket:`8141` 

.. versionadded:: 1.4.0b1

.. _change_4926:

4926 - sql: Add support for division operators (// and /) to numeric types
--------------------------------------------------------------------------

Implemented full support for "truediv" and "floordiv" using the "/" and "//"
operators.  A "truediv" operation between two expressions using
  :class:`_types.Integer`  now considers the result to be   :class:` _types.Numeric` ,
and the dialect-level compilation will cast the right operand to a numeric type on
a dialect-specific basis to ensure truediv is achieved.  For floordiv, conversion is
also added for those databases that don't already do floordiv by default (MySQL,
Oracle) and the ``FLOOR()`` function is rendered in this case, as well as for cases
where the right operand is not an integer (needed for PostgreSQL, others). The change
resolves issues both with inconsistent behavior of the division operator on different
backends and also fixes an issue where integer division on Oracle would fail to be
able to fetch a result due to inappropriate outputtypehandlers.

Related tickets:  :ticket:`4926` 

.. versionadded:: 1.4.0b1

.. _change_5465:

5465 - oracle: Implement binary precision for FLOAT datatype, reflect "binary_precision" as schema-level info
------------------------------------------------------------------------------------------------------------

Implemented DDL and reflection support for ``FLOAT`` datatypes which include an
explicit "binary_precision" value. Using the Oracle-specific   :class:`_oracle.FLOAT` 
datatype, the new parameter  :paramref:`_oracle.FLOAT.binary_precision`  may be
specified which will render Oracle's precision for floating point types directly.
This value is interpreted during reflection. Upon reflecting back a ``FLOAT`` datatype,
the datatype returned is one of   :class:`_types.DOUBLE_PRECISION`  for a ` `FLOAT`` for
a precision of 126 (this is also Oracle's default precision for ``FLOAT``),
  :class:`_types.REAL`  for a precision of 63, and   :class:` _oracle.FLOAT`  for a custom
precision, as per Oracle documentation.

As part of this change, the generic  :paramref:`_sqltypes.Float.precision`  value is
explicitly rejected when generating DDL for Oracle, as this precision cannot be
accurately converted to "binary precision"; instead, an error message encourages the
use of  :meth:`_sqltypes.TypeEngine.with_variant`  so that Oracle's specific form of
precision may be chosen exactly. This is a backwards-incompatible change in behavior,
as the previous "precision" value was silently ignored for Oracle.

Related tickets:  :ticket:`5465` 

.. versionadded:: 1.4.0b1

.. _change_7086:

7086 - postgresql: Use plainto_tsquery() for text searching match operation
----------------------------------------------------------------------------

The  :meth:`Operators.match`  operator now uses ` `plainto_tsquery()`` for PostgreSQL
full text search, rather than ``to_tsquery()``. The rationale for this change is to
provide better cross-compatibility with match on other database backends. Full support
for all PostgreSQL full text functions remains available through the use of  :data:`.func` 
in conjunction with  :meth:`Operators.bool_op`  (an improved version of  :meth:` Operators.op` 
for boolean operators).

Related tickets:  :ticket:`7086` 

.. versionadded:: 1.4.0b1

.. _change_5052:

5052 - sql: Improved ISO-8601 rendering when using literal binds
---------------------------------------------------------------

Added modified ISO-8601 rendering (i.e. ISO-8601 with the T converted to a
space) when using ``literal_binds`` with the SQL compilers provided by the
PostgreSQL, MySQL, MariaDB, MSSQL, Oracle dialects. For Oracle, the ISO
format is wrapped inside of an appropriate TO_DATE() function call. Previously
this rendering was not implemented for dialect-specific compilation.

Related tickets:  :ticket:`5052` 

.. versionadded:: 1.4.0b1

.. _change_7258:

7258 - deprecations: Remove legacy dialect packages
---------------------------------------------------

Removed the firebird, informix and maxdb dialects that were previously deprecated.

Related tickets:  :ticket:`7258` 

.. versionchanged:: 1.4.0b1

.. _change_7744:

7744 - orm: Improve binary expression construction to avoid recursion
----------------------------------------------------------------------

Improved the construction of SQL binary expressions to allow for very long
expressions against the same associative operator without special steps needed in
order to avoid high memory use and excess recursion depth. A particular binary
operation ``A op B`` can now be joined against another element ``op C`` and the
resulting structure will be "flattened" so that the representation as well as SQL
compilation does not require recursion.

One effect of this change is that string concatenation expressions which use SQL
functions come out as "flat", e.g. MySQL will now render
``concat('x', 'y', 'z', ...)``` rather than nesting together two-element functions
like ``concat(concat('x', 'y'), 'z')``.  Third-party dialects which override the
string concatenation operator will need to implement a new method
``def visit_concat_op_expression_clauselist()`` to accompany the existing
``def visit_concat_op_binary()`` method.

Related tickets:  :ticket:`7744` 

.. versionadded:: 1.4.7


.. _change_5469:

5469 - postgresql, mysql: Add double types to namespace.
---------------------------------------------------------

Added   :class:`.Double` ,   :class:` .DOUBLE` ,
  :class:`_sqltypes.DOUBLE_PRECISION` 
datatypes to the base ``sqlalchemy.`` module namespace, for explicit use of
double/double precision as well as generic "double" datatypes. Use
  :class:`.Double`  for generic support that will resolve to DOUBLE/DOUBLE
PRECISION/FLOAT as needed for different backends.

.. versionadded:: 1.4.0

.. _change_8216:

8216 - postgresql: Add JSONPATH type for cast expressions
--------------------------------------------------------

Introduced the type   :class:`_postgresql.JSONPATH`  that can be used
in cast expressions. This is required by some PostgreSQL dialects
when using functions such as ``jsonb_path_exists`` or ``jsonb_path_match``
that accept a ``jsonpath`` as input.

Related tickets:  :ticket:`8216` 

.. seealso::

      :ref:`postgresql_json_types`  - PostgreSQL JSON types.

.. versionadded:: 1.4.8

.. _change_4038:

4038 - mysql, mariadb: Support Partitioning
--------------------------------------------

Add support for Partitioning and Sample pages on MySQL and MariaDB reflected options.
The options are stored in the table dialect options dictionary, so
the following keyword need to be prefixed with ``mysql_`` or ``mariadb_``
depending on the backend.
Supported options are:

* ``stats_sample_pages``
* ``partition_by``
* ``partitions``
* ``subpartition_by``

These options are also reflected when loading a table from database,
and will populate the table  :attr:`_schema.Table.dialect_options` .
Pull request courtesy of Ramon Will.

Related tickets:  :ticket:`4038` 

.. versionadded:: 1.4.11

.. _change_8288:

8288 - mssql: Reflect clustered index flag
-------------------------------------------

Implemented reflection of the "clustered index" flag ``mssql_clustered``
for the SQL Server dialect. Pull request courtesy John Lennox.

Related tickets:  :ticket:`8288` 

.. versionadded:: 1.4.11

.. _change_7442:

7442 - postgresql: Reflect expression-based indexes
----------------------------------------------------

The PostgreSQL dialect now supports reflection of expression based indexes.
The reflection is supported both when using
  :meth:`_engine.Inspector.get_indexes`   and when reflecting a
  :class:`_schema.Table`  using  :paramref:` _schema.Table.autoload_with` .

Related tickets:  :ticket:`7442` 

.. versionadded:: 1.4.18

.. _change_7471:

7471 - sql: Add anonymous alias for ambiguous names in FROM clause
-----------------------------------------------------------------

Added an additional lookup step to the compiler which will track all FROM
clauses which are tables, that may have the same name shared in multiple schemas
where one of the schemas is the implicit "default" schema; in this case, the table
name when referring to that name without a schema qualification will be rendered
with an anonymous alias name at the compiler level in order to disambiguate the two
(or more) names. The approach of schema-qualifying the normally unqualified name
with the server-detected "default schema name" value was also considered, however this
approach doesn't apply to Oracle nor is it accepted by SQL Server, nor would it work
with multiple entries in the PostgreSQL search path. The name collision issue resolved
here has been identified as affecting at least Oracle, PostgreSQL, SQL Server, MySQL and
MariaDB.

Related tickets:  :ticket:`7471` 

.. versionadded:: 1.4.18

.. _change_6962:

6962 - engine: Remove implicit_returning parameter from create_engine()
-----------------------------------------------------------------------

The  :paramref:`_sa.create_engine.implicit_returning`  parameter is deprecated on the
  :func:`_sa.create_engine`  function only; the parameter remains available on the
  :class:`_schema.Table`  object. This parameter was originally intended to enable the
"implicit returning" feature of SQLAlchemy when it was first developed and was not
enabled by default. Under modern use, there's no reason this parameter should be
disabled, and it has been observed to cause confusion as it degrades performance and
makes it more difficult for the ORM to retrieve recently inserted server defaults.
The parameter remains available on   :class:`_schema.Table`  to specifically suit
database-level edge cases which make RETURNING infeasible, the sole example currently
being SQL Server's limitation that INSERT RETURNING may not be used on a table that
has INSERT triggers on it.

Related tickets:  :ticket:`6962` 

.. versionchanged:: 1.4.18

.. _change_6962_oracle:

6962 - oracle: implicit-returning feature enabled in all cases
-------------------------------------------------------------

Related to the deprecation for  :paramref:`_sa.create_engine.implicit_returning` ,
the "implicit_returning" feature is now enabled for the Oracle dialect in all cases;
previously, the feature would be turned off when an Oracle 8/8i version were detected,
however online documentation indicates both versions support the same RETURNING syntax
as modern versions.

.. versionchanged:: 1.4.18

.. _change_8102:

8102 - orm: Remove warnings about reflection of constraints when columns are excluded
-------------------------------------------------------------------------------------

The warnings that are emitted regarding reflection of indexes or unique constraints, when
the  :paramref:`.Table.include_columns`  parameter is used to exclude columns that are
then found to be part of those constraints, have been removed. When the :paramref:`.Table.
include_columns` parameter is used it should be expected that the resulting
  :class:`.Table`  construct will not include constraints that rely upon omitted columns.
This change was made in response to  :ticket:`8100`  which repaired :paramref:` .Table.
include_columns` in conjunction with foreign key constraints that rely upon omitted columns,
where the use case became clear that omitting such constraints should be expected.

Related tickets:  :ticket:`8102` 

.. versionadded:: 1.4.18

.. _change_7495:

7495 - orm: Do not defer primary key or polymorphic discriminator column
--------------------------------------------------------------------------

The behavior of   :func:`_orm.defer`  regarding primary key and "polymorphic
discriminator" columns is revised such that these columns are no longer
deferrable, either explicitly or when using a wildcard such as ``defer('*')``.
Previously, a wildcard deferral would not load PK/polymorphic columns which led to
errors in all cases, as the ORM relies upon these columns to produce object identities.
The behavior of explicit deferral of primary key columns is unchanged as these
deferrals already were implicitly ignored.

Related tickets:  :ticket:`7495` 

.. versionadded:: 1.4.18

.. _change_6980:

6980 - typing: New variant method returns full type
---------------------------------------------------

The  :meth:`_sqltypes.TypeEngine.with_variant`  method now returns a copy of
the original   :class:`_sqltypes.TypeEngine`  object, rather than wrapping it
inside the ``Variant`` class, which is effectively removed (the import symbol remains
for backwards compatibility with code that may be testing for this symbol). While the
previous approach maintained in-Python behaviors, maintaining the original type allows
for clearer type checking and debugging.

  :meth:`_sqltypes.TypeEngine.with_variant`   also accepts multiple dialect names per call
as well, in particular this is helpful for related backend names such as ``"mysql",
"mariadb"``.

Related tickets:  :ticket:`6980` 

.. versionadded:: 1.4.18SQLite datetime、date 和 time 数据类型已使用 Python 标准库中的 fromisoformat() 方法来解析传入的 datetime、date 和 time 字符串值。与以前基于正则表达式的方法相比，这种方法提高了性能，也自动适应了包含六位“毫秒”格式或三位“毫秒”格式的 datetime 和 time 格式。

.. change::
    :tags: usecase, mssql
    :tickets: 7844

    MSSQL 现在支持在创建表时添加表和列注释。还新增了反射表注释的支持。感谢 Daniel Hall 在此 pull request 中提供的帮助。

.. change::
    :tags: mssql, removed
    :tickets: 7258

    移除了 mxodbc 驱动程序的支持，因为缺乏测试支持。ODBC 用户可以使用完全受支持的 pyodbc 方言。

.. change::
    :tags: mysql, removed
    :tickets: 7258

    移除了针对 MySQL 和 MariaDB 的 OurSQL 驱动程序的支持，因为此驱动程序似乎未得到维护。

.. change::
    :tags: postgresql, removed
    :tickets: 7258

    移除了多个废弃的驱动程序的支持:

        - 对于 PostgreSQL 的 pypostgresql。这个驱动程序可以作为外部驱动程序使用，而不是内置于 SQLAlchemy 中的 https://github.com/PyGreSQL
        - 对于 PostgreSQL 的 pygresql。

    请切换到其中一个受支持的驱动程序或同一驱动程序的外部版本。

.. change::
    :tags: bug, engine
    :tickets: 7953

    修复了  :meth:`.Result.columns`  方法中的问题，其中使用单个索引调用  :meth:` .Result.columns`  在某些情况下，特别是 ORM 结果对象中，可能导致   :class:`.Result`  返回标量对象而不是   :class:` .Row`  对象，就像调用了  :meth:`.Result.scalars`  方法一样。在 SQLAlchemy1.4 中，这种情况会发出警告，指出它的行为将在 SQLAlchemy2.0 中更改。

.. change::
    :tags: usecase, sql
    :tickets: 7759

    向  :meth:`.HasCTE.add_cte`  添加了一个新的参数  :paramref:` .HasCTE.add_cte.nest_here` ，它将一个给定的   :class:`.CTE` “嵌套”到父语句的级别。此参数等效于使用  :paramref:` .HasCTE.cte.nesting`  参数，但在某些情况下可能更直观，因为它允许同时设置嵌套属性和 CTE 的显式级别。

     :meth:`.HasCTE.add_cte`  方法还接受多个 CTE 对象。

.. change::
    :tags: bug, orm
    :tickets: 7438

    修复了  :paramref:`._orm.Mapper.eager_defaults`  参数的行为，以便在仅使用表定义中的客户端端 SQL 默认值或 onupdate 表达式时，ORM 向插入或更新操作发出时将触发使用 RETURNING 或 SELECT 来获取行的操作，此前仅使用表 DDL 部分和/或服务器端 onupdate 表达式时才会触发此获取，即使在渲染获取时也会包含客户端端 SQL 表达式。

.. change::
    :tags: performance, schema
    :tickets: 4379

    改进了模式反射 API 的架构，以允许参与的方言使用高性能的批量查询，通过减少一倍的查询次数来反映许多表的模式。新的性能功能首先针对 PostgreSQL 和 Oracle 后端，可以应用于使用对系统目录表进行 SELECT 查询以反映表的任何方言。此更改还包括   :class:`.Inspector`  对象的新 API 功能和行为改进，包括像  :meth:` .Inspector.has_table` 、  :meth:`.Inspector.get_table_names`   这样的方法的一致、缓存行为以及新方法  :meth:` .Inspector.has_schema`  和  :meth:`.Inspector.has_index` 。

    .. seealso::

          :ref:`change_4379`  - 全部背景

.. change::
    :tags: bug, engine

    将   :class:`.DefaultGenerator`  对象（如   :class:` .Sequence` ）传递给  :meth:`.Connection.execute`  方法已过时，因为此方法被声明为返回一个   :class:` .CursorResult`  对象，而不是一个普通标量值。应使用  :meth:`.Connection.scalar`  方法，该方法已重新制定了新的内部代码路径，以适合使用 SELECT 调用默认生成对象，而不必通过  :meth:` .Connection.execute`  方法。

.. change::
    :tags: usecase, sqlite
    :tickets: 7185

    SQLite 方言现在支持 UPDATE..FROM 语法，用于 UPDATE 语句。语句的 WHERE 标准可能会引用未在子查询中使用的其他表中的表，而无需使用子查询。当使用   :class:`_dml.Update`  构造时使用多个表或其他实体或可选择项时，会自动调用此语法。

.. change::
    :tags: general, changed

     :meth:`_orm.Query.instances`  方法已弃用。该方法的行为合同，即可以通过使用诸如 :meth` .Select.from_statement` 或   :func:`_orm.aliased`  的构造将对象迭代到任意结果集中，已经过时并不再被测试。可以使用任意语句返回对象，例如  :meth:` _sql.literal` 。

.. change::
    :tags: feature, orm

    使用包含   :class:`_schema.ForeignKey`  引用的   :class:` _schema.Column`  对象的声明式 mixin 不再需要使用   :func:`_orm.declared_attr`  来实现此映射。当将列应用于已声明的映射时，将   :class:` _schema.ForeignKey`  对象与   :class:`_schema.Column`  本身一起复制。

.. change::
    :tags: oracle, feature
    :tickets: 6245

    在 cx_Oracle 方言中实现了完整的“RETURNING”支持，覆盖了两个单独的功能类型:

    * 实现了多行 RETURNING，这意味着对于产生多个 RETURNING 行的 DML 语句，现在可以接收多个 RETURNING 行。
    * 也实现了“executemany RETURNING”- 这允许在使用 ``cursor.executemany()`` 时，RETURNING 在每个语句返回一行。与像是上次在 SQLAlchemy1.4 中添加到 psycopg2 中的方式一样，此功能的实现通过将 ORM 插入的性能显着提高。

.. change::
    :tags: oracle

    cx_Oracle 7 现在是 cx_Oracle 的最低版本。

.. change::
    :tags: bug, sql
    :tickets: 7551

    对于从值类型确定 SQL 类型的 Python 字符串值，主要是在使用   :func:`_sql.literal`  时，现在会应用   :class:` _types.String`  类型，而不是采用以前的全部使用   :class:`_types.Unicode`  数据类型。如果字符串不是 ` `isascii()``，则仍将绑定   :class:`_types.Unicode`  数据类型，以前在所有字符串检测中都使用。这种行为**仅适用于在使用 ` `literal()`` 或其他上下文中进行就地数据类型检测的情况**，在正常的   :class:`_schema.Column`  比较操作下，被比较的   :class:` _schema.Column`  的类型始终是优先考虑的。

    使用   :class:`_types.Unicode`  可以确定像 SQL Server 这样的后端的文字字符串格式，其中一个文字值（即使用 ` `literal_binds``）将呈现为 ``N'<value>'`` 而不是 ``'value'``。对于正常的绑定值处理，  :class:`_types.Unicode`  数据类型对于将值传递给 DBAPI 也可能具有影响，在 SQL Server 的情况下，pyodbc 驱动程序支持使用   :ref:` setinputsizes mode <mssql_pyodbc_setinputsizes>` ，这将处理   :class:`_types.String`  和   :class:` _types.Unicode`  的差异。

.. change::
    :tags: bug, sql
    :tickets: 7083

      :class:`_functions.array_agg`  现在将数组维度设置为 1。改进了   :class:` _types.ARRAY`  处理，以接受多数组的值作为多数组的 ``None`` 值。