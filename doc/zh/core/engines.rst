.. _engines_toplevel:

====================
引擎配置
====================

:class:`_engine.Engine`是任何SQLAlchemy应用程序的起点。
它是通过连接池和一个:class:`.Dialect`向SQLAlchemy应用程序提供实际数据库及其:term:`DBAPI`的“home base”，
这个:Dialect映射了如何与特定类型的数据库 / DBAPI组合进行通信。

总体结构可以如下图所示：

.. image:: sqla_engine_arch.png

如上图，:class:`_engine.Engine`引用了一个:class:`.Dialect`和一个:class:`_pool.Pool`，
两者一起解释DBAPI的模块函数以及数据库的行为。

创建引擎只需要发出一个调用，:func:`_sa.create_engine()`::

    from sqlalchemy import create_engine

    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost:5432/mydatabase")

上面的引擎创建了一个针对PostgreSQL的:class:`.Dialect`对象，
以及一个:class:`_pool.Pool`对象，该对象在首次接收到连接请求时将在``localhost:5432``建立DBAPI连接。
请注意，:class:`_engine.Engine`及其底层:class:`_pool.Pool`直到调用:meth:`_engine.Engine.connect`方法，
或调用依赖于此方法的操作，例如:meth:`_engine.Engine.execute`才建立第一个实际的DBAPI连接。
通过这种方式，:class:`_engine.Engine`和:class:`_pool.Pool`可以被认为具有惰性初始化行为。

创建后的:class:`_engine.Engine`可以直接用于与数据库进行交互，
也可以传递给:class:`.Session`对象以与ORM一起使用。本节讲述配置:class:`_engine.Engine`的详细信息。
下一节，:ref:`connections_toplevel`，将详细介绍:class:`_engine.Engine`和类似的使用API，通常用于非ORM应用程序。

.. _supported_dbapis:

支持的Database
===================

SQLAlchemy包括许多:class:`.Dialect`的实现，适用于各种后端。其中大多数常见数据库的方言包含在SQLAlchemy中；
少数其他数据库需要安装单独的方言。

有关各种后端的详细信息，请参见:ref:`dialect_toplevel`部分。

.. _database_urls:

Database URLs
=============

:func:`_sa.create_engine`函数基于URL生成:class:`_engine.Engine`对象。URL的格式通常遵循 `RFC-1738 <https://www.ietf.org/rfc/rfc1738.txt>`_，其中
下划线，而不是破折号或句点，在“scheme”部分中被接受。
URL通常包括用户名，密码，主机名，数据库名称字段，以及用于附加配置的可选关键字参数。
在某些情况下，可以接受文件路径，而在其他情况下，“数据源名称”取代了“主机” 和“数据库”部分。
数据库URL的典型格式为：

.. sourcecode:: text

    dialect+driver://username:password@host:port/database

方言名称包括SQLAlchemy方言的标识名称，例如``sqlite``，``mysql``，``postgresql``，``oracle``或``mssql``。
驱动程序名称是DBAPI的名称，用所有小写字母连接到一起。如果未指定，则默认情况下将导入一个“地图”DBAPI，如果可用，则此默认设置通常是可以得到它的后端的最广泛的驱动程序。

转义密码中的特殊字符，例如@字符
----------------------------------------------------------

当构造完整的URL字符串以传递给:func:`_sa.create_engine`时，
**需要对特殊字符（例如在用户和密码中使用的字符）进行URL编码才能正确解析。**.
**这包括@符号。**

以下是包含密码“kx@jj5/g”的URL的示例，
其中“at”符号和斜线字符分别表示为“％40”和“％2F”：

.. sourcecode:: text

    postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb


可以使用`urllib.parse <https://docs.python.org/3/library/urllib.parse.html>`_生成上述密码的编码：

  >>> import urllib.parse
  >>> urllib.parse.quote_plus("kx@jj5/g")
  'kx%40jj5%2Fg'

然后可以将URL作为字符串传递给:func:`_sa.create_engine`::

    from sqlalchemy import create_engine

    engine = create_engine("postgresql+pg8000://dbuser:kx%40jj5%2Fg@pghost10/appdb")

作为生成完整的URL字符串的替代方法，
传递给:func:`_sa.create_engine`的对象可以是:class:`.URL`实例，
它绕过了解析阶段，并可以直接容纳未转义的字符串。
见下一节的示例

.. versionchanged:: 1.4

    修复了主机名和数据库名称中的``@``标记符的支持。 由于此修复的副作用，
    必须转义密码中的``@``号。

动态生成身份验证令牌
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:meth:`.DialectEvents.do_connect`还是动态插入身份验证令牌
可能会在整个:class:`_engine.Engine`的使用寿命内发生更改。
例如，如果令牌由``get_authentication_token()``生成并在``token``参数中传递给DBAPI，则可以通过以下方式实现：

    from sqlalchemy import event

    engine = create_engine("postgresql+psycopg2://user:pass@hostname/dbname")


    @event.listens_for(engine, "do_connect")
    def provide_token(dialect, conn_rec, cargs, cparams):
        cparams["token"] = get_authentication_token()


隐藏参数
------------------

:class:`_engine.Engine`发出的日志还指示了该语句的一些SQL参数摘录。
为了保护隐私，防止这些参数被记录下来，启用:paramref:`_sa.create_engine.hide_parameters`标志::

    >>> e = create_engine("sqlite://", echo=True, hide_parameters=True)>>> with e.connect() as conn:
...     conn.execute(text("select :some_private_name"), {"some_private_name": "pii"})
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine select ?
2020-10-24 12:48:32,808 INFO sqlalchemy.engine.Engine [由于hide_parameters=True而隐藏的SQL参数]