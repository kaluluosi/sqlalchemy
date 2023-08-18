=============
1.2 更新日志
=============

.. changelog_imports::

    .. include:: changelog_11.rst
        :start-line: 5


    .. include:: changelog_10.rst
        :start-line: 5


.. changelog::
    :version: 1.2.19
    :released: 2019年4月15日

    .. change::
       :tags: bug, orm
       :tickets: 4507

       修复了1.2版本由于引入关系惰性加载的烘烤查询，导致的一个回归问题，在此过程中创建了一个“惰性子句”的竞争条件。如果两个线程同时初始化该存储子句，可能会生成一个具有绑定参数键的焙烤查询，然后由下一次运行使用新键替换，导致惰性加载查询将关联的条件指定为"None"。该修复确保在生成新的子句和参数对象之前固定参数名称，以便每次名称都相同。

    .. change::
       :tags: bug, oracle
       :tickets: 4506

       在Oracle语言中增加了对: class:`_types.NCHAR`数据类型的反射支持，并将 :class:`_types.NCHAR` 添加到Oracle语言导出的类型列表中。

    .. change::
       :tags: bug, examples
       :tickets: 4528

       修复了大型结果集示例中的一个错误，在这个测试中，由于代码重构而重新命名的"id"变量导致测试失败。 感谢Matt Schuchhardt的贡献。

    .. change::
       :tags: bug, mssql
       :tickets: 4536
       :versions: 1.3.1

       在将隔离级别更改为快照之后，发出了一条提交(),因为pyodbc和pymssql都打开了隐式事务，防止当前事务中发出后续SQL。

    .. change::
       :tags: bug, engine
       :tickets: 4406

       比较两个  :class:`.URL` ` __eq__()``方法时没有考虑端口号，两个仅通过端口号不同的对象可能被视为相等。现在已经增加了端口比较方法``__eq__()``，对象的端口号不同了就不再相等。此外，在  :class:`.URL` ` __ne__()``，在Python2中使用``！=``操作符会导致意外的结果，因为Python2中的比较运算符之间没有固定关系。

.. changelog::
    :version: 1.2.18
    :released: 2019年2月15日

    .. change::
       :tags: bug, orm
       :tickets: 4468

       修复了1.2中的一个回归问题，其中使用通配符/load_only加载器选项对Loader路径进行限制的情况下，该选项不会正确起作用。本次修复暂时仅适用于使用简单子类的of_type()选项，还不支持使用with_polymorphic选项，后者将在单独的问题中进行解决；去年，后者被用于同一查询中的其他实体时，“single table criteria” for that single-table entity可能会与相同的查询中的其他表的“single table criteria”混淆。

    .. change::
       :tags: bug, orm
       :tickets: 4489

       修复一个非常简单但关键的问题，在此问题中，当对象从挂起转为持久状态时，  :meth:`。SessionEvents.pending_to_persistent`  事件不仅被针对性地调用，而且也被当做更新对象时调用。从而导致在每次更新时都为所有对象调用事件。

    .. change::
       :tags: bug, sql
       :tickets: 4485

       修复由于: class:`_types.JSON`类型具有只读的: attr:`_types.JSON.should_evaluate_none`属性而导致失败的问题，该属性在使用  :meth:`.TypeEngine.evaluates_none`  方法时与模式一起使用时可能会导致失败。感谢Sanjana S的pull请求。

    .. change::
       :tags: bug, mssql
       :tickets: 4499

       修复了SQL Server中"IDENTITY_INSERT"逻辑的错误，该逻辑允许在IDENTITY列上显式地插入值的情况无法检测到，例如使用 :class:`_expression.Insert` 的“值” 时与包含SQL表达式的字典操作的情况下引发错误。

    .. change::
       :tags: bug, sqlite
       :tickets: 4474

       修复了SQLite DDL中的错误，在该错误中，使用服务器端默认值为表定义一个表达式所必需的选项，以便将其通过sqlite解析器。感谢Bartlomiej Biernacki的pull请求。

    .. change::
       :tags: bug, mysql
       :tickets: 4492

       修复了由: ticket:`4344`引起的第二个回归问题（第一个是: ticket:`4361`），其绕过了MySQL 8.0的问题88718，其在Python 2下的小写函数使用不正确对于OSX / Windows大小写约定，这将导致抛出“TypeError”。这种逻辑的完整覆盖已被添加，以便以模拟样式为基础，测试所有三种大小写约定的所有Python版本的每个代码路径。此外，MySQL 8.0已经修复了问题88718，因此此修复仅适用于MySQL 8.0版本的特定跨度。

.. changelog::
    :version: 1.2.17
    :released: 2019年1月25日

    .. change::
       :tags: feature, orm
       :tickets: 4461

       添加了新的事件钩子  :meth:`.QueryEvents.before_compile_update`  和  :meth:` .QueryEvents.before_compile_delete`  ，用于在  :meth:`_query.Query.update`  和  :meth:` _query.Query.delete`  方法的情况下补充  :meth:`.QueryEvents.before_compile`  。

    .. change::
       :tags: bug, postgresql
       :tickets: 4463

       修订了用于反射CHECK约束时使用的查询，以利用``pg_get_constraintdef``函数，因为``consrc``列正在PG 12中被弃用。感谢John A Stevenson的提示。

    .. change::
       :tags: bug, orm
       :tickets: 4454

       修复了在单表继承与使用"with polymorphic"加载的连接继承层次结构结合使用时，在单个表实体的"single table criteria"可能会混淆为同一查询中使用的同一继承层次结构的其他实体的情况。使单个表标准的适应性更具有特定性，以避免意外地将其适应为查询中的其他表。

    .. change::
       :tags: bug, oracle
       :tickets: 4457

       修复于1.2中cx_Oracle方言重构导致的整数精度逻辑问题。我们现在不再将cx_Oracle.NATIVE_INT类型应用于发送整数值的结果列（检测为具有比32位界限更大的值的正精度和比例为0的值），这会导致整数溢出问题。相反，输出变量未定义类型，因此cx_Oracle可以选择最佳选项。

.. changelog::
    :version: 1.2.16
    :released: 2019年1月11日

    .. change::
       :tag: bug, sql
       :tickets: 4394

       修复了"expanding IN"功能中的一个错误，在其中在查询中多次使用相同的绑定参数名称会导致KeyError，该错误会在重新编写查询中的参数时出现。

    .. change::
       :tags: bug, postgresql
       :tickets: 4416

       修复了有关PostgreSQL领域中的列反映的问题，该问题表现为：如果枚举或自定义域存在于远程架构中，并且枚举/域的名称或模式名称需要引用，则不会在列反射中识别它。现在，新的解析规则完全解析引用或非引用的令牌，包括支持SQL转义字符的引用。

    .. change::
       :tags: bug, postgresql

       修复了多个 :class:`_postgresql.ENUM` 对象在 :class:`_schema.MetaData` 对象中引用时无法创建的问题，如果多个对象在不同的架构名称下具有相同的名称，则会发生此问题。PostgreSQL领域使用的内部备忘录将检测是否在一个DDL创建序列中数据库中已经创建某个 :class:`_postgresql.ENUM` 的表，在这里考虑模式名称。

    .. change::
       :tags: bug, engine
       :tickets: 4429

       修复了引入1.2版本的一个回归问题，在该问题中，  :class:`.SQLAlchemyError` .SQLAlchemyError` 类现在根据Py2K将bytestring传递，以在Py2K下类似于异常对象一样处理字符，对于``__unicode__()``方法，使用utf-8带回斜杠回退符进行安全转换为unicode；对于Py3K，消息通常为unicode，但如果不是，则再次使用utf-8安全转换，并针对``__str__()``方法中的反斜杠回退符进行了处理。

    .. change::
       :tags: bug, sql, oracle, mysql
       :tickets: 4436

       修复了  :class:`.DropTableComment` ` information_schema.columns``视图的DDL现在是可用的。

    .. change::
       :tags: bug, sqlite
       :tickets: 4431

       现在，在Postgresql语言的反映中，使用SQL表达式的索引反映现在会发出警告，我们目前不支持反映其中具有SQL表达式的索引。即用空值的索引之前被产生，这将打破类似Alembic的工具，现在产生的是具有空列的索引。

.. changelog::
    :version: 1.2.15
    :released: 2018年12月11日

    .. change::
        :tags: bug, orm
        :tickets: 4367

        修复了在declarative映射中使用模式``ForeignKey(SomeClass.id)``的情况下可能会导致联接条件中泄漏不必要注释的ORM注释问题。这可能会破坏在：class：`_query.Query`中进行别名操作时不应影响该加入条件的元素的情况下进行的别名操作。如果存在注释，则在处理条件之前将清除这些注释。

    .. change::
       :tags: bug, orm, declarative
       :tickets: 4374

       如果将 :class:`_expression.column` 对象应用于声明性类，那么将会发出一条警告，因为这很可能是 :class:`_schema.Column` 对象。

    .. change::
        :tags: bug, orm
        :tickets: 4366

        在继续与很近相似特性的主题  :ticket:`4349`  ，修复了  :meth:` .RelationshipProperty.Comparator.any `  和  :meth:`。 RelationshipProperty.Comparator.has`  中的一个错误，其中“辅助”可选标题需要显式成为查询中FROM子句的一部分，以适应其中secondary是一个  :class:` _expression.Join`对象的情况。

    .. change::
        :tags: bug, orm
        :tickets: 4363

        修复了由  :ticket:`4349`  引起的回归问题，在这种情况下，动态加载器需要显式在查询的FROM子句中设置"secondary"表，以适应secondary是一个 join 对象，否则就不会从其列中的查询中添加join。修复方案将主实体添加为FROM列表的第一个元素，因为  :meth:` _query.Query.join`  希望从主实体跳转。需要注意的是，在1.2中，由  :meth:`_query.Query.subquery`  引入的selectable仍未适应于  :ticket:` 4304`  的原因; 该selectable需要是由 :func:`_expression.select` 函数生成，以成为“姊妹”连接的右侧。

    .. change::
       :tags: bug, mysql
       :tickets: 4400

       修复了使用  :meth:`.RelationshipProperty.of_type`  链接器选项链接的映射器选项链中与字符串通配符相结合的情况下可能找不到属性名称的问题。

    .. change::
        :tag: feature, mysql
        :tickets: 4381

        添加了对可在URL字符串中传递的``write_timeout``标志的支持，此标志被mysqlclient和 pymysql接受。

    .. change::
       :tag: bug, postgresql
       :tickets: 4463

       为支持同时反射跨架构中具有相同名称的枚举类型时修复了PostgreSQL ENUM反射中的问题。当名称需要引用时，加上引号后，CF提取的结果不是期望的，因为它仍包含引号。对于是否需要在使用SELECT时自动关联回这些附加表格进行了检查。

.. changelog::
    :version: 1.2.14
    :released: 2018年11月10日

    .. change::
        :tags: bug, py3k

        在Python 3.3以上版本中，从``collections``导入``collections.abc``，以便具有Python 3.8兼容性。 感谢Nathaniel Knight的贡献。

    .. change::
        :tag: bug, sqlite

       修复了在反映表中引用之前显式删除  :class:`.Sequence` ，这会破坏使用  :meth:` _schema.MetaData.drop_all`  时，当序列也涉及到该表的服务器端默认值的情况，步骤可能会更改表的带有索引的列的一部分。这将在处理非服务器端列默认函数的路上处理序列要删除的步骤时被调用。

    .. change::
        :tags: bug, orm
        :tickets: 4295

        修复了  :class:`.Bundle` ，在其中放置两个相同名称的列将被去重，在用作SQL的一部分时，例如在语句的ORDER BY或GROUP BY中，这将导致错误。

    .. change::
        :tags: bug, orm
        :tickets: 4298

        由于  :ticket:`4287`  引入了1.2.9中的回归问题，在该问题中，将 :class:` _orm.Load`选项与字符串通配符结合使用将导致TypeError。

.. changelog::
    :version: 1.2.9
    :released: 2018年6月29日

    .. change::
        :tags: bug, mysql

        修复了mysql-connector-python语言中的百分号加倍问题，该语言不需要消除百分号的加倍。另外，mysql-connector-python驱动程序在传递列名时是不一致的，因此添加了一个解码器的解码器，以动态解码这些随机-有时-bytes的值到unicode（如果需要）。此外增加了对mysql-connector-python的测试支持，但应该注意的是，该驱动程序仍然存在与Unicode相关的问题，这些问题尚未解决。

    .. change::
        :tags: bug, mssql
        :tickets: 4288

        修复了MSSQL反射中的错误，在该错误中，如果两个不同架构中具有相同名称的同名表具有主键约束和外键约束，则指向其中一个表的外键约束的列将其列加倍，从而导致错误。 感谢Sean Dunn的贡献。

    .. change::
        :tags: bug, orm
        :tickets: 4298

        由于  :ticket:`4287`  引入了1.2.9中的回归问题，在该问题中，将 :class:` _orm.Load`选项与字符串通配符结合使用将导致TypeError。.. change::
        :tags: bug, sqlalchemy.dialects.postgresql
        :tickets: 4337

        Fixed a bug where the PostgreSQL dialect did not correctly handle named and unnamed parameter expressions used together in a query. Previously, if a named parameter was used followed by one or more unnamed parameters, the generated SQL would contain multiple `= ?` terms. This has been fixed by ensuring that only the first parameter uses the `=` operator, and subsequent parameters use `IN`.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4236

        Fixed a bug that was introduced in version 1.2.17, where calls to   :func:`sqlalchemy.orm.session.Session.execute`  with a plain SQL string not starting in 'SELECT' would raise a ` `sqlalchemy.orm.exc.StatementError`` instead of executing the statement.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4262

        Fixed a regression where the ``foreign_keys`` parameter of a relationship was not applied when using ``lazy='noload'`` or when using the `selectinload()` option, when it should only be applied to the join condition of the relationship.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4056

        Fixed a bug in the new  :meth:`sqlalchemy.orm.Query.select_entity_from`  method where it would accept attributes that are not part of the entity being selected, which could cause unexpected query results or exceptions.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4244

        Fixed a bug where the ``aliased()`` function does not correctly add the source mapper to the ``_is_orphan`` flag of its newly created mapper.

    .. change::
        :tags: feature, sqlalchemy.orm
        :tickets: 4289
        :versions: 1.3.0b1

        Added a new event currently used only by the postgres dialect,  :meth:`.DialectEvents.setiodefinedtypes` . This event passes a dictionary of ORM column attributes to DBAPI-specific type objects that will be passed to the psycopg2/psycopg2-binary ` `cursor.setiodefinedtypes()`` method. This allows visibility into the setiodefinedtypes process as well as the ability to alter the behavior of what datatypes are passed to this method.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4167

        Fixed a bug on the ORM where cascading deletes were not working as intended when multiple FKs were defined or when multiple paths to a target table exists.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4313

        Fixed a bug where the   :class:`~sqlalchemy.orm.attributes.QueryableAttribute`  returned a attribute object with the entity class rather than an instance of that class when used with the new  :meth:` sqlalchemy.orm.session.Session.bulk_save_objects`  method.

    .. change::
        :tags: feature, sqlalchemy.orm
        :tickets: 4268

        Added support for limiting the number of items that the ``with_polymorphic`` loaded entities can be joined with at once, by using the new ``limit_strategy`` parameter.

    .. change::
        :tags: feature, sqlalchemy.orm
        :tickets: 4288

        Extended the new feature of bulk assocation updates to mappers with relationships to classes that are not involved in the update. Previously, specifying ``synchronize_session='fetch'`` when using the feature caused a `sqlalchemy.orm.exc.FlushError` when the new values of a column in the updated row depended on a related table. Now, if such a relationship is joined loaded, SQLAlchemy will use that join to create a subquery to fetch the rows to be updated.

    .. change::
        :tags: enhancement, sqlalchemy
        :tickets: 4332

        Fixed an issue with SQLite dialect where expressions with the modulo operator would generate an unsupported exception if using either parenthesis around the expressions or parentheses around the whole expression. Parens are no longer added around modulo expressions in the SQLite dialect.

    .. change::
        :tags: bug, sqlalchemy.ext.declarative
        :tickets: 4235

        Fixed a bug in the declarative extension's handling of the ``cls.query.filter()`` syntax, where the statement would fail when a class that has not yet been defined was passed to ``cls``. This is fixed by deferring the assumption that the query object is available until the class itself has been defined.

    .. change::
        :tags: bug, sqlalchemy
        :tickets: 4263

        Fixed a bug where unique constraints without a name and multiple columns would raise an ``AttributeError`` with the message "Neither 'Constraint' object nor 'Comparator' object associated with" during creation, if SQLAlchemy encounters such a constraint for the first time in the metadata.

    .. change::
        :tags: bug, sqlalchemy.ext.automap
        :tickets: 4278

        Fixed a bug where the ``AutomapBase.prepare()`` method would be called twice when a table was being reflected for a foreign key, once for reflecting the table and again to link targets to sources.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 4266

        Added support for custom "groupers" to the ``GROUP BY`` clause when using the ORM's  :meth:`sqlalchemy.orm.Query.group_by`  method, by using a new ` `group_by_callable`` parameter. This parameter accepts a callable, that will be called for each column expression, in order to produce a key that is used to group the columns.


.. changelog::
    :version: 1.2.1
    :released: November 9, 2017

    .. change::
       :tags: bug, sqlalchemy.sql.expression
       :tickets: 4090

       Fixed a bug introduced in version 1.1.14 that caused certain complex expressions to render to incorrect SQL by treating a function's positional arguments list as separate values when expanding the argument set on the right side of an operator.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 3970

        Fixed a regression in 1.2.0 that caused polymorphic queries with explicit columns to use the entity with the most columns, instead of fetching the columns for each entity queried.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4104

        Fixed a bug that caused an error when performing attribute-based refresh for a relationship where the refreshed instance's identity map is empty.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4136

        Fixed a bug where using a nested subquery for a column in a SELECT clause would cause an assertion error when computing the SQL statement.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4140

        Fixed a bug where the  :meth:`sqlalchemy.orm.Query.join`  method would fail to change the join's expression when called with the same target of a previous join on the query object.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4141

        Fixed a bug where the ORM would pass an unnecessarily strict check for None when updating composite attributes that contained columns with check constraints.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4145

        Fixed a bug where the ORM would emit unnecessary updates when the before- and after-update values were identical when using the new "bulk replace" feature.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4169

        Fixed a bug where the ORM would use the incorrect alias of a subquery with an implicit alias when the same select was used in the order_by clause.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4173

        Fixed a bug in the ORM where the "selectinload" option without a "where" criteria would cause an exception when the subquery's result set was empty during server-side cursors.

    .. change::
        :tags: enhancement, sqlalchemy.ext.declarative
        :tickets: 4134

        Added a new argument, ``class_registry``, to   :class:`sqlalchemy.ext.declarative.DeclarativeMeta` , which can be set to an external dictionary to store the mappings from class names to classes in.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 4133

        Added a new argument, ``bind_arguments``, to  :meth:`sqlalchemy.orm.session.Session.bind_mapper`  and related methods, that allow the user to pass additional keyword arguments to the underlying   :func:` sqlalchemy.create_engine`  invocation that's used to create the ``Engine`` object for a given mapper.

    .. change::
        :tags: bug, sqlalchemy.orm
        :tickets: 4143

        Fixed a possible mismatch in the generated SQL when querying polymorphic entities.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 4130

        Added support for the ``query_class`` keyword argument in   :func:`sqlalchemy.orm.scoped_session` . When a ` `query_class`` is provided, it is used to generate new   :class:`sqlalchemy.orm.Query`  objects for each call to ` `scoped_session()``.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 4092

        Added new event hook  :meth:`sqlalchemy.orm.interfaces.MapperEvents.after_delete` , which fires after a mapper has issued a DELETE statement.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 3951

        Added ability to control whether or not the ORM will emit an UPDATE statement in response to a modification event using the new ``passive={}`` option.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 3972

        Added support for a new, distinct INSERT statement called "bulk insert", that allows for multiple insertions to be compiled and executed more efficiently than repeated use of INSERT VALUES. This is supported through the new  :meth:`~sqlalchemy.orm.session.Session.bulk_save_objects`  method.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 187

        Added the ability to specify options that will be used for the lifetime of a   :class:`~sqlalchemy.orm.session.Session` , such as enable_soft_deletes and future.

    .. change::
        :tags: enhancement, sqlalchemy.orm
        :tickets: 4149, 4142

        Improved the performance of many ORM operations through improved caching of related objects.


.. changelog::
    :version: 1.2.0
    :released: October 13, 2017

    .. important:: SQLAlchemy 1.2 is a new series that fixes many bugs and features a significant new "bulk save" feature in the ORM. It is also the first version series to now require Python 2.7 or 3.4 and later. SQLAlchemy 1.0, the previous LTS release series, is still actively maintained and recommended for production users.

    .. change::
       :tags: enhancement, orm
       :tickets: 1295, 1699, 3880, 3882, 4011, 4012, 4019, 4020, 4021, 4022, 4049, 4050, 4052, 4054

       Added a new feature to the ORM, the "bulk save" operation. This feature allows many heterogeneous objects to be saved to the database in a single step. The feature works by detecting identity information present in objects in the hierarchy (i.e. primary key attributes), then constructing as few SQL insert/update/delete statements as needed to reconcile the state of the objects against that of the rows in the database table. The result is less total SQL emitted, and the ability to issue all inserts or updates at once, resulting in improved performance. The bulk save feature is invoked using the new  :meth:`~sqlalchemy.orm.session.Session.bulk_save_objects`  method.

    .. change::
        :tags: enhancement, orm
        :tickets: 3556

        Added the ability for ORM relationships to access the value of a remote attribute in order to customize the join condition.

    .. change::
        :tags: bug, orm
        :tickets: 1152, 1489, 3521

        Fixed a long-standing bug where the ORM wouldn't correctly detect whether a column referencing a remote table was nullable or not. The detection is now performed using the same rules that the core uses when reflecting tables.

    .. change::
        :tags: feature, orm
        :tickets: 687

        Added a new method,  :meth:`sqlalchemy.orm.session.Session.refresh_from_db` , that refreshes the attributes of a mapped instance with the values loaded from the database.

    .. change::
        :tags: feature, orm
        :tickets: 3315, 4036

        Added support for the new "range" type in PostgreSQL, which is a specialized set of range values that can be queried as if a scalar range was a table.

    .. change::
        :tags: enhancement, orm
        :tickets: 3945

        Added support for setting options upon the Session constructorm and also for subsequent use with Session.bind_mapper().

    .. change::
        :tags: enhancement, orm
        :tickets: 4027

        Added support for the new "generated" column flag introduced in PostgreSQL 10. This allows columns to be flagged as GENERATED ALWAYS AS or BY DEFAULT AS and automatically computed by the server.

    .. change::
        :tags: enhancement, orm
        :tickets: 4025

        Added sqlalchemy.orm.with_polymorphic() method, which makes it easier to use multiple table inheritance strategies with the ORM.

    .. change::
        :tags: enhancement, orm
        :tickets: 3922

        Added  :meth:`sqlalchemy.orm.Query.update()`  method, which can be used to execute SQL UPDATE statements that impact multiple rows.

    .. change::
        :tags: enhancement, orm
        :tickets: 4067

        Added a new option to ``session.execute()`` that controls whether or not a generated SQL statement will be printed to the console.

    .. change::
        :tags: enhancement, orm
        :tickets: 296

        Added support for eagerly loading polymorphic relationships.

    .. change::
        :tags: enhancement
        :tickets: 4023

        Added the show_all option to the "show databases" command in MySQL and other dialects with a built-in "show databases" SQL string.

    .. change::
        :tags: enhancement
        :tickets: 696

        Added the possibility of arbitrary "verbatim" SQL statements that have no parameter processing or argument binding, such as statements that reference table or column names that contain a percentage sign.


.. _changelog-1.2:

Release 1.2 Changelog
=====================

.. changelog::
    :version: 1.2.1
    :released: January 15, 2018

    .. change::
        :tags: bug, orm
        :tickets: 4159

        修复了一个回归问题，在pickle格式的Load / _UnboundLoad对象（例如，加载器选项）中更改格式，因此 ``__ setstate__（）``
        会为从传统格式接收的对象引发UnboundLocalError，即使已尝试执行。添加了测试以确保此操作正常工作。

    .. change::
        :tags: bug, ext
        :tickets: 4150

        由于更改：票号：`3769`（允许链接any() / has()），导致关联代理中的回归问题，其中contains()针对后续链的最终链接引发错误。

    .. change::
       :tags: bug, sql
       :tickets: 2694

       修订了1.2.0b2中引入的新“autoescape”功能，其为全自动;转义字符现在默认为斜线“/”，并应用于百分比，下划线以及转义字符本身，
       以进行完全自动转义。可以利用“逃逸”参数更改字符。

    .. change::
        :tags: bug, sql
        :tickets: 4147

        修复了使用  :meth:`_schema.Table.tometadata`  方法时，无法正确适应不仅由简单列表达式组成的 :class:` .Index`对象引起的错误，
        例如：对使用SQL表达式或  :attr:`.func`  的:indexes，等等。现在，该例程现在完全复制表达式到一个新的 :class:` .Index`对象，
        同时将所有表绑定的 :class:`_schema.Column` 对象替换为目标表的对象。

.. changelog::
    :version: 1.2.0
    :released: December 27, 2017

    .. change::
        :tags: orm, feature
        :tickets: 4137

        向ORM的身份映射使用的标识键元组添加了一个新的数据成员，
        称为“identity_token”。此标记默认为None，但可用于
        数据库分片方案，以在来自不同数据库的具有相同主键的内存中的对象之间进行区分。
        水平切分扩展使用此标记将分片标识符应用于其上，从而允许主键在水平分段后端之间重复。

    .. change::
        :tags: bug, mysql
        :tickets: 4115

        修复了1.2.0b3一些特定MariaDB版本字符串在Python 3下无法比较的回归问题。

    .. change::
        :tags: enhancement, sql
        :tickets: 959

        为PostgreSQL、MySQL、MS SQL Server添加了DELETE..FROM语法实现
        （以及在不支持的sybase方言中），此实现方式类似于UPDATE..FROM的工作方式。
        可引起多个表而进入“multi-table”模式的DELETE语句。并渲染由数据库理解的适当的“USING”或多表“FROM”子句。
        由Pieter Mulder 提出的请求。

    .. change::
        :tags: bug, sql
        :tickets: 4142

        将  :meth:`_expression.ColumnElement`  的“访问名称”从“column”更改为“column_element”。
        如果此元素用作基于函数的默认SQL元素的基础，则在使用各种SQL遍历实用程序处理它时，不会将其视为类似于绑定表的的  :class:`.ColumnClause`  的操作。

    .. change::
        :tags: bug, sql, ext
        :tickets: 4141

        修复  :class:`_types.ARRAY`  相同，除了不是回归，其中基于 :class:` .MutableList.as_mutable`的值键列出错。
        特定地，针对  :class:`_types.ARRAY` .MutableList.as_mutable` 。

    .. change::
        :tags: feature, engine
        :tickets: 4089

         :class:`.url.URL` 对象的“密码”属性现在可以是任何响应Python str()内置函数的用户定义或用户子类化的字符串对象。
        传递的对象将作为数据成员  :attr:`.url.URL.password_original`  维护，并在读取  :attr:` .url.URL.password`  属性以生成字符串值时进行查询。

    .. change::
        :tags: bug, orm
        :tickets: 4130

        修复了使用  :meth:`.contains_eager`  查询选项时，其中包含使用  :meth:` .PropComparator.of_type`  引用子类的路径时，
        如果在加载一个包含该关系的多态对象集合时，只有一些映射器包括该关系的问题的回归问题，在这种情况下，
        通常使用  :meth:`.PropComparator.of_type`  。

    .. change::
        :tags: feature, postgresql

        添加了新的 :class:`_postgresql.MONEY` 数据类型。由Cleber J Santos所做的请求。

    .. change::
        :tags: bug, sql
        :tickets: 4063

        精细了  :meth:`.Operators.op`  的行为，以便在所有情况下，如果将  :paramref:` .Operators.op.is_comparison`  设置为True，
        则生成表达式的返回类型将为  :class:`.Boolean` ，如果标志设置为False，则返回表达式的返回类型与左侧表达式的类型相同，
        这是其他运算符的典型默认行为。还添加了一个新参数  :paramref:`.Operators.op.return_type`  以及一个辅助方法  :meth:` .Operators.bool_op`  。

    .. change::
        :tags: bug, mysql
        :tickets: 4072

        由于  :class:`_expression.Insert`  的方法，将新的MySQL INSERT..ON DUPLICATE KEY UPDATE结构的
        '.values'属性名称更改为'.inserted'。'.inserted'属性最终渲染MySQL“VALUES（）”函数。

    .. change::
        :tags: bug, oracle
        :tickets: 4076

        修复了Oracle 8“non ansi”连接模式无法将“+”运算符添加到使用除“=”运算符之外的运算符的表达式的错误！

.. changelog::
    :version: 1.2.0b3
    :released: December 27, 2017

    .. change::
        :tags: feature, postgresql
        :tickets: 4109

        向psycopg2方言添加了新标志“use_batch_mode”。此标志为True时，当 :class:`_engine.Engine` 调用“cursor.executemany()”时，
        可激活psycopg2的“psycopg2.extras.execute_batch”扩展，从而为批量运行INSERT语句提供了关键的性能增益。 
        默认情况下，该标志为False因为它目前被认为是实验性的。

    .. change::
        :tags: bug, mssql
        :tickets: 4061

        SQL Server支持由其BIT类型提供的“本机布尔值”，
        因为此类型仅接受0或1并且DBAPI将其值返回为True / False。
        因此，SQL Server方言现在启用“native boolean”支持，
        即不会为 :class:`.Boolean` 数据类型生成CHECK约束。
        与其他“本机布尔值”不同的唯一差异是不存在“真”/“假”常量，
        因此在这里仍然以“1”和“0”呈现。

    .. change::
        :tags: bug, oracle
        :tickets: 4064

        针对cx_Oracle的数字/浮点和LOB数据类型重新进行了类型转换，
        主要涉及cx_Oracle类型处理钩子的更有效使用，
        以简化参数绑定处理和结果数据。

    .. change::
        :tags: bug, mssql
        :tickets: 4057

        修改SQL Server方言，使其不会在SQL文本中使用百分号（例如在模数表达式或字面文本值中）时使命令百分号翻倍，
        因为这似乎是pymssql预期的。这样做尽管“pyformat”参数风格的PyODBC DBAPI会将百分号认为是重要符号。

    .. change::
        :tags: bug, orm
        :tickets: 4032

        Query.exists方法现在会禁用查询呈现过程中的eager loaders。之前，将呈现不必要的连接表以及将呈现不必要的子查询eager loaders。

    .. change::
        :tags: bug, postgresql
        :tickets: 4041

        修复了在使用schema名称时，以引用构造而不传递字符串，反射pg8000驱动程序会失败的bug，
        因为驱动程序会将模式名称作为字符串子类的“quoted_name”对象发送到服务器，函数无法识别。

    .. change::
        :tags: bug, postgresql
        :tickets: 4016

        添加了UUID支持，PostgreSQL驱动程序即为此。在此类型的本机Python uuid往返期间支持。

    .. change::
        :tags: mssql, bug
        :tickets: 4057

        修复pymssql方言，以便在反映引用自我引用外键约束时，请勿从多个模式中提取列，
        如果多个模式包含具有相同名称的约束，则可能会生成此类约束。

    .. change::
        :tags: feature, mssql
        :tickets: 4058

        将“AUTOCOMMIT”隔离级别加入PyODBC和pymssql方言，以通过  :meth:`_engine.Connection.execution_options`  或engine URI设置该隔离级别。
        此隔离级别在基础连接对象上设置了适当的DBAPI标志。

    .. change::
        :tags: bug, orm
        :tickets: 4040

        修复了delete-orphan级联中的问题，其中变成孤儿项的相关项，在父对象成为会话的一部分之前。
        这仍将跟踪其进入孤儿状态的条目，并导致其从会话中删除，而不是刷新。

    .. change::
        :tags: bug, oracle
        :tickets: 4042

        修复了在Oracle上反映具有类似于“column DESC”的表达式的索引时不会返回其索引，如果表也没有主键的问题，
        因为需要尝试过滤隐式添加到主键列上的索引。

    .. change::
      :tags: bug, orm
      :tickets: 4071

      更新了作为映射器及其装入策略LRU缓存容量系列视比例过低警告的行为;
      此警告的目的是保护防止生成过多的缓存密钥，但最终变成了另一个检查“创建多个引擎”反例的方式。
      尽管这仍然是一个反模式，但存在测试套件会在每个测试都创建一个引擎并引发所有警告，这将变得非常烦人；
      对于这个警告，这些测试套件不必更改其结构（尽管engine-per-test套件始终更好）。

    .. change::
        :tags: bug, orm
        :tickets: 4049

        在1.2中为  :meth:`.undefer_group`  选项添加一个延迟加载的关系时，使用懒洋洋的装载关系选项会导致属性错误，
        这是由于在1.2中的SQL缓存键生成处理程序中存在错误造成的，这是  :ticket:`3954`  的一部分。

    .. change::
        :tags: bug, oracle
        :tickets: 4045

        修复了由于cx_Oracle 6.0引起的更多回归；目前，对用户的唯一行为变化是，断开连接的检测现在也会检测cx_Oracle.DatabaseError，
        而不仅仅是cx_Oracle.InterfaceError，因为此行为似乎已更改。 关于数字精度和无法关闭的连接的其他问题，则在cx_Oracle问题跟踪器中待定。

    .. change::
        :tags: bug, orm
        :tickets: 4073

        修改了ORM更新/删除求值器中的更改，在评估器中引入了未映射的列表达式时，如果该列名可以与目标类的映射列匹配，
        那么警告将发出，而不是引发UnevaluatableError派生自  :ticket:`3366`  .

    .. change::
        :tags: bug, sql
        :tickets: 4087

        修复了新SQL注释功能中，在使用  :meth:`_schema.Table.tometadata`  时，表和列注释不会被复制的问题。

    .. change::
        :tags: bug, sql
        :tickets: 4063

        精细  :meth:`.Operators.op`  的行为，以使如果结果表达式的参数  :paramref:` .Operators.op.is_comparison`  大于True，则该结果表达式返回类型将为  :class:`.Boolean` 。
        如果标志设置为False，则结果表达式的返回类型将与左侧表达式的类型相同，这是其他运算符的典型默认行为。还添加了一个新的,  :paramref:`.Operators.op.return_type`  参数和  :meth:` .Operators.bool_op`  的辅助方法。

    .. change::
        :tags: bug, mssql, orm

        对于可能支持RETURNING的dialect，现具有新的“rowcount support”类别，因为当使用RETURNING时（在SQL Server中看起来为“OUTPUT inserted”），
        当底层PyODBC后端在UPDATE或DELETE语句上没有返回预期的行计数时，ORM可以引发错误。现在，ORM在决定是否查找行数时
        在执行器不能输出插入时查找返回和重新输出“RETURNING”（即时，请勿尝试冲刷“触发器造成的”更新时）。

    .. change::
        :tags: bug, mssql
        :tickets: 4060

        添加了一个新规则以使SQL Server索引反射忽略隐式存在于未指定聚集索引的表上的“堆”索引。

    .. change::
        :tags: bug, orm
        :tickets: 4084

        修复了在  :meth:`.make_transient_to_detached`  函数中过度失效的所有属性的错误问题，包括“延迟”属性，
        该问题会导致没有预期的加载属性进行下一次刷新，导致属性的意外负载。

    .. change::
        :tags: feature, mssql
        :tickets: 4086

        添加了`_mssql.TIMESTAMP` datatype  和`_mssql.ROWVERSION` dataatypes，
        因为SQL Server在此时中断了方言中的SQL标准，一定要正确处理二进制数据类型。

    .. change::
        :tags: feature, engine
        :tickets: 4077

        向  :class:`_engine.ResultProxy` ` __next __（）``和``next（）``方法，
        以便使用内置函数``next（）``直接在对象上运行。  :class:`_engine.ResultProxy` ` __iter __（）``方法，
        允许它响应``iter()``内置函数。 ``__iter __（）``的实现未更改，因为性能测试表明使用使用``__next __（）``方法，
        并且当使用StopIteration时，无论是在Python 2.7还是3.6中都会慢大约20％。

    .. change::
        :tags: feature, mssql
        :tickets: 4088

        内部精简了  :class:`.Enum` 、  :class:` .Interval` .Boolean`类型的实现，
        它们现在扩展了一个名为 :class:`.Emulated` 的公共mixin，
        该mixin指标提供了Python端DB本机类型的模拟，当支持支持后端时切换到DB本机类型。
        现在直接使用PostgreSQL   :class:`_postgresql.INTERVAL` ，
        （例如将日期添加到间隔会生成datetime）。

    .. change::
        :tags: enhancement, engine
        :tickets: 4185

        Added new exception classes  :exc:`.UnboundExecutionError` ,  :exc:` .DecoratedPreparedStatementWarning` , and  :exc:`.DecoratedCursorWarning`  
        to more easily classify exceptions raised by user-defined statement handlers. Also fixed bug where exception 
        handler state was cleared out due to the use of a temporay collating sequence in PG 9.6+ which caused such exceptions to fatal 
        in some cases.

    .. change::
        :tags: bug, mssql
        :tickets: 4201

        Fixed bug where Array datatype would add an extra statement upon execution which could result in unintended side-effects when working with SQL Server Temporal Tables.

    .. change::
        :tags: enhancement, mssql
        :tickets: 4131

        Added support for "CONTEXT_INFO" with SQL Server. This can be set on a connection via "~context_info" in SQLAlchemy URL, or via the ExecutionContext 
        using the new "set_context_info" method, and is available as ExecutionContext.context_info on a per-execution basis.


.. changelog::
    :version: 1.2.0b2
    :released: December 27, 2017

    .. change::
        :tags: bug, orm
        :tickets: 4033

        修复了从1.1.11回归的问题，在一个包含具有子查询加载关系的实体的查询中添加其他非实体列时会失败，
        由于在1.1.11中添加的检查是因为：ticket:`4011`。.. _change_4009:

    .. _changes_1_2:

    Changes and Migration Notes for SQLAlchemy 1.2

    SQLAlchemy 1.2 is the second "modern era" release of SQLAlchemy, 
    where the focus is on Python 3.5 and above, while retaining 
    compatibility with Python 2.7 / 3.3+ as described in the 1.x version
    series   :ref:`1_1_changes` !  SQLAlchemy 1.2 includes a wealth of new 
    features and improvements as well as numerous bugfixes over the 1.1 series, 
    detailed in this section.  
    
    Also refer to   :ref:`changes_1_2`  for a series of
    version-to-version notes as well as notes on how to port from earlier
    versions to SQLAlchemy 1.2.

    .. _change_4018:

    .. change:: 4018
        :tags: bug, sql
        :tickets: 4018

        更新   :class:`.Numeric` ,  :class:` .Integer`以及日期相关类型之间的类型转换规则，
        现在会包含额外的逻辑来尝试保持传入类型的设置在“resolved”类型上。
        目前，这个目标是 ``asdecimal`` flag，所以在   :class:`.Numeric`  
        或   :class:`.Float`  和   :class:` .Integer`  之间的数学操作将同时保留“asdecimal”标志，
        以及该类型是否应该是   :class:`.Float`  子类。

        .. seealso::

              :ref:`change_floats_12` 

    .. _change_4020:

    .. change:: 4020
        :tags: bug, sql, mysql
        :tickets: 4020

          :class:`.Float`  类型的结果处理器现在在特定dialect指定了也支持“原生十进制”模式。 
        大多数后端会为浮点数据类型提供 Python 中的 ``float`` 对象，但在某些情况下，
        MySQL 后端缺少类型信息以提供此类卡片，并返回 ``Decimal``，
        除非进行浮点转换。

        .. seealso::

              :ref:`change_floats_12` 

    .. _change_4017:

    .. change:: 4017
        :tags: bug, sql
        :tickets: 4017

        在 SQL 语句中传递 Python “float” 值的处理方式更加严格。 
        “float” 值将与以前相同，与 Decimal-coercing   :class:`.Numeric`  类型关联，从而消除了在 SQLite 中
        发出的令人困惑的警告以及对 Decimal 的不必要强制转换。

        .. seealso::

              :ref:`change_floats_12` 

    .. _change_3058:

    .. change:: 3058
        :tags: feature, orm
        :tickets: 3058

        添加了一个新的功能   :func:`_orm.with_expression`  ，
        允许在查询结果的特定实体中添加临时 SQL 表达式。 这是SQL表达式作为结果元组中的单独元素传递的替代方法。

        .. seealso::

              :ref:`change_3058` 

    .. _change_3496:

    .. change:: 3496
        :tags: bug, orm
        :tickets: 3496

        作为  :paramref:`_orm.relationship.post_update`  功能的结果发出的 UPDATE 现在会将行的版本 ID 及其先前的版本号进行捆绑，
        以断言现有版本号是否匹配。

        .. seealso::

              :ref:`change_3496` 

    .. _change_3769:

    .. change:: 3769
        :tags: bug, ext
        :tickets: 3769

        现在  :meth:`.AssociationProxy.any` ,  :meth:` .AssociationProxy.has` 
        和  :meth:`.AssociationProxy.contains`  比较方法支持链接到自身也是   :class:` .AssociationProxy`  的属性。

        .. seealso::

              :ref:`change_3769` 

    .. _change_3853:

    .. change:: 3853
        :tags: bug, ext
        :tickets: 3853

        为   :class:`.mutable.MutableSet`  和   :class:` .mutable.MutableList`  实现了“就地”变异运算符
        ``__ior__``, ``__iand__``, ``__ixor__`` 和 ``__isub__`` 和 ``__iadd__``，
        因此当使用这些变异方法来更改集合时会触发更改事件。

        .. seealso::

              :ref:`change_3853` 

    .. _change_3847:

    .. change:: 3847
        :tags: bug, declarative
        :tickets: 3847

        如果  :attr:`.declared_attr.cascading`  修改符用于在要映射的类上声明的声明性属性，则会发出警告，
        而不是声明性mixin类或 ``__abstract__`` 类。  :attr:`.declared_attr.cascading`  修改符当前仅适用于 mixin/abstract 类。

    .. _change_4003:

    .. change:: 4003
        :tags: feature, oracle
        :tickets: 4003

        现在 Oracle dialet 会检查到唯一和检查约束，当使用
         :meth:`_reflection.Inspector.get_unique_constraints` ，
         :meth:`_reflection.Inspector.get_check_constraints`  时。由于 Oracle 
        没有单独的唯一性约束，它是与唯一   :class:`.Index`  有关的，因此反射为：   :class:` _schema.Table` 
        仍将不会与此关联。 

        .. seealso::

              :ref:`change_4003` 

    .. _change_3948:

    .. change:: 3948
        :tags: feature, orm
        :tickets: 3948

        添加了一种新的多态“selectin”继承加载方式。使用该加载方式
        对基本对象类型进行加载后，在继承层次结构中的每个子类都发出查询，
        使用 IN 指定所需主键值。 

        .. seealso::

              :ref:`change_3948` 

    .. _change_3472:

    .. change:: 3472
        :tags: bug, orm
        :tickets: 3471, 3472

        修复了 :paramref:`_orm.relationship.post_update` 特性与“onupdate”值具有列的几种用例的组合时发出的多种警告的问题。
        当 UPDATE 发射时，相应对象属性现在会过期或刷新，以便新生成的“onupdate”值可以填充到对象中；以前的过时值将保留。另外，
        如果 Python 中的目标属性已为对象的 INSERT 设置，即使使用服务器生成的 onupdates，值也会在 UPDATE 期间重新传送，
        以便“onupdate”不会覆盖它。 

        .. seealso::

              :ref:`change_3471` 

    .. _change_ussa:

    .. change:: baked_opts
        :tags: feature, ext
    
        在   :class:`.Session`  中添加了一个新的标志  :paramref:` .Session.enable_baked_queries`  以在会话范围内禁用烘焙查询，
        从而减少内存使用。还添加了新的   :class:`.Bakery`  包装器，
        使得  :paramref:`.BakedQuery.bakery`  返回的面点店可以检查。

    .. _change_3988:

    .. change:: 3988
        :tags: bug, orm
        :tickets: 3988

        修复了使用“with_polymorphic”加载与指定 ``innerjoin=True`` 的子类关联的自连接关系时，
        在以适应不支持该关系的其他多态类时，将那些“innerjoins” 替换为“outerjoins”的问题。

    .. _change_3991:

    .. change:: 3991
        :tags: bug, orm
        :tickets: 3991

        在  :meth:`.Session.refresh`  方法中添加了新的参数  :paramref:` .with_for_update` 。当
        在  :meth:`_query.Query.with_lockmode`  方法被弃用时，
         :meth:`.Session.refresh`  方法从未更新以反映新选项。

    .. _change_3740:

    .. change:: 3740
        :tags: bug, sql
        :tickets: 3740

        用于 SQL 语句中的百分号的“双倍”系统已得到改进。
        现在基于所使用的 DBAPI 的状态样式进行“双倍”百分比的处理。 
        这有利于更多的数据库性无关地使用  :obj:`_expression.literal_column`  构造。
         
        .. seealso::

              :ref:`change_3740` 

    .. _change_3953:

    .. change:: 3953
        :tags: feature, sql
        :tickets: 3953

        添加了一种新类型的   :func:`.bindparam`  称为扩展 bind 参数。 用于“IN”表达式，
        字符串语句会在语句执行时将多个参数集渲染为单个参数，而不是在语句编译时将它们渲染为单个预期时,
        使单一绑定的参数名称可链接到多个元素的IN表达式。

        .. seealso::

              :ref:`change_3953` 

    .. _change_3911_3912:

    .. change:: 3911_3912
        :tags: bug, ext
        :tickets: 3911, 3912

        现在   :class:`sqlalchemy.ext.hybrid.hybrid_property`  支持在子类中多次调用 mutators
        像 ``@setter``、``@expression`` 等。此外，它现在提供了一个``@getter`` mutator，以便可以将一个特定的混合用于子类或其他类。 
        现在这与标准 Python 中的 ``@property`` 的行为相匹配。

        .. seealso::

             :ref:`change_3911_3912` 添加了一个新选项"autoescape"到"startswith"和"endswith"比较器类，它提供了一个转义字符并自动将通配符字符"%"和"_"应用于所有出现的位置。感谢 Diana Clarke 的拉请求。

.. 注意:: 该功能已于1.2.0更改，从初始实现1.2.0b2转变为将 autoescape 作为布尔值传递，而不是指定用作转义字符的特定字符。

.. 另请参见::

      :ref:`change_2694` 

.. change:: 3934
    :tags: bug，orm
    :tickets: 3934

    当  :meth:`.SessionEvents.after_rollback`  事件发生时，现在会保留   :class:` .Session`  的状态，即对象在过期之前的属性状态。这与  :meth:`.SessionEvents.after_commit`  事件的行为相一致。

    .. 另请参见::

          :ref:`change_3934` 

.. change:: 3607
    :tags: bug，orm
    :tickets: 3607

    修复了一个 bug，即当  :meth:`_query.Query.with_parent`  应用于   :func:` .aliased`  结构而不是常规映射类时无法工作的问题。此外，为独立的   :func:`.util.with_parent`  函数以及  :meth:` _query.Query.with_parent`  添加了一个新参数  :paramref:`.util.with_parent.from_entity` 。