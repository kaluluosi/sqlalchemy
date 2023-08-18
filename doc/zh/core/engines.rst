.. _engines_toplevel:

====================
引擎配置
====================

  :class:`_engine.Engine` .Dialect` 通过连接池和  :class:`.Dialect` ，用于区分术语和说明文本的方式解释数据库的行为。这些应用程序描述了与特定类型的数据库/DBAPI组合谈话的方法。

一般结构如下所示：

.. image:: sqla_engine_arch.png

上面，  :class:`_engine.Engine` .Dialect` 和一个  :class:`_pool.Pool` ，它们在一起解释了DBAPI的模​​块函数以及数据库的行为。

创建引擎只涉及发出单个调用，  :func:`_sa.create_engine()` ::

    from sqlalchemy import create_engine

    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost:5432/mydatabase")

上面的引擎创建了一个专为PostgreSQL量身定制的  :class:`.Dialect` ` localhost：5432``建立一个DBAPI连接。请注意，  :class:`_engine.Engine`  方法或依赖于此方法的操作（例如  :meth:` _engine.Engine.execute`  ）之前建立第一个实际的DBAPI连接。通过这种方式，可以说 :class:`_engine.Engine` 和 :class:`_pool.Pool` 具有*懒初始化*行为。

一旦创建  :class:`_engine.Engine` ，可以直接使用它与数据库交互，或者将其传递给  :class:` .Session` ,将详细介绍 :class:`_engine.Engine` 及类似对象的使用API，通常用于非ORM应用程序。

.. _supported_dbapis:

支持的数据库
===================

SQLAlchemy包括许多 :class:`.Dialect` 实现，用于不同的后端。与大多数常见的数据库相关的方言已包含在SQLAlchemy中；另外一些需要额外安装一个单独的方言。

请参见 :ref:`dialect_toplevel` 一节，了解各种后端的详细信息。

.. _database_urls:

数据库URLs
=============

 :func:`_sa.create_engine` 函数基于URL生成 :class:`_engine.Engine` 对象。URL的格式通常遵循`RFC-1738<https://www.ietf.org/rfc/rfc1738.txt>`_，除了在“方案”部分之内接受下划线而不是短横线或句点。URL通常包括用户名、密码、主机名、数据库名称字段，以及可选的关键字参数用于其他配置。在某些情况下，文件路径是可接受的，而在其他情况下，“数据源名称”替换“主机”和“数据库”部分。数据库URL的典型形式为：

.. sourcecode:: text

    dialect+driver://username:password@host:port/database

方言名称包括SQLAlchemy方言的标识名称，例如 ``sqlite``，``mysql``，``postgresql``，``oracle``或``mssql``的名称。driverName是用于使用所有小写字母连接到数据库的DBAPI的名称。如果未指定，则会导入“默认”DBAPI（如果可用）-这个默认值通常是该后端可用的最通用的驱动程序。

转义密码中的特殊字符，如@符号
----------------------------------------------------------

当构建一个完整的URL字符串传递给 :func:`_sa.create_engine` 时，**特殊字符，例如在用户和密码中使用的字符需要进行URL编码，以便被正确解析**。**这就包括了“@”符号**。

以下是一个URL示例，其中包括密码“kx@jj5/g”，其中“at”符号和斜杆字符分别表示为“%40”和“%2F”：

.. sourcecode:: text

    postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb


上述密码的转码可以使用`urllib.parse <https://docs.python.org/3/library/urllib.parse.html>`_生成::

  >>> import urllib.parse
  >>> urllib.parse.quote_plus("kx@jj5/g")
  'kx%40jj5%2Fg'

然后可以将URL作为字符串传递给  :func:`_sa.create_engine` ::

    from sqlalchemy import create_engine

    engine = create_engine("postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb")

作为转义特殊字符以创建完整的URL字符串的替代方法，传递给  :func:`_sa.create_engine` .URL` 类的实例，该实例绕过解析阶段，直接适应未转义的字符串。有关示例，请参见以下部分。

.. versionchanged:: 1.4

    修复了主机名和数据库名中的``@``符号的支持。作为此修复的副作用，必须转义密码中的``@``符号。

动态生成身份验证令牌
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_engine.Engine` .DialectEvents.do_connect` 事件挂钩来全面接管连接过程，从而为生命周期内可能会更改的令牌提供理想的方法。例如，如果标记通过` `get_authentication_token()``生成并在一个``token``参数中传递给DBAPI，那么可以实现如下：

    from sqlalchemy import event

    engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")


    @event.listens_for(engine, "do_connect")
    def provide_token(dialect, conn_rec, cargs, cparams):
        cparams["token"] = get_authentication_token()

.. seealso::

      :ref:`mssql_pyodbc_access_tokens`  - 一个涉及SQL Server的更具体的例子

屏蔽参数
------------------

  :class:`_engine.Engine`  标志即可：

    e = create_engine("sqlite://", echo=True, hide_parameters=True)

>>> with e.connect() as conn:
...     conn.execute(text("select :some_private_name"), {"some_private_name": "pii"})
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine select ?
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine [由于hide_parameters=True参数被隐藏, SQL参数没有显示]