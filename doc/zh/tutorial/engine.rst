.. |prev| replace:: :doc:`index`
.. |next| replace:: :doc:`dbapi_transactions`

.. include:: tutorial_nav_include.rst

.. rst-class:: core-header, orm-addin

.. _tutorial_engine:

建立连接 - 引擎
==========================================

.. container:: orm-header

    **欢迎ORM和Core的读者们！**

    每个连接数据库的SQLAlchemy应用程序都需要使用一个 :class:`_engine.Engine`。<br>
     本小节供所有人阅读。

任何SQLAlchemy应用程序的起点都是一个名为 :class:`_engine.Engine` 的对象。<br>
该对象作为到特定数据库的连接的中心源，提供了一个工厂，以及用于这些数据库连接的一个称为 :ref:`connection pool <pooling_toplevel>` 的存储空间。<br>
引擎通常是针对特定数据库服务器创建的全局对象，使用URL字符串进行配置，该字符串描述了如何连接到数据库主机或后端。

在本教程中，我们将使用一个仅位于内存中的SQLite数据库。<br>
这是一种无需在实际上预先存在的数据库中进行测试的简单方法。<br>
:class:`_engine.Engine` 是通过使用 :func:`_sa.create_engine` 函数创建的：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)

:class:`_sa.create_engine` 的主要参数是一个字符串URL，以上代码传递一个字符串 “sqlite+pysqlite:///:memory:” 作为参数。<br>
此字符串向 :class:`_engine.Engine` 指示以下三个重要的事实：

1. 我们要与什么类型的数据库通信？这是以上的 `sqlite` 部分，将 SQLAlchemy 链接到一种被称为 :term:‘dialect‘ 的对象。

2. 我们使用了哪个 :term:‘DBAPI‘？Python的 :term:‘DBAPI‘ 是 SQLAlchemy 用来与特定数据库交互的第三方驱动程序。在本例中，我们使用名称 “pysqlite”，这在现代 Python 中是 SQLite 的标准库接口。如果省略，则 SQLAlchemy 将使用特定于所选择的数据库的默认 :term:‘DBAPI‘。

3. 我们如何定位数据库？在本例中，我们的URL包括短语“/:memory:”，这是一个指示 “sqlite3” 模块我们将使用仅位于内存中的数据库的指示符。这种类型的数据库非常适合进行测试，因为它不需要服务器，也不需要创建新文件。

.. sidebar:: 惰性连接

    :class:`_engine.Engine` 在首次函数调用 :func:`_sa.create_engine` 返回时没有尝试实际连接到数据库，仅在其第一次试图针对数据库执行任务时才会发生。<br>
    这是一种称为 :term:‘lazy initialization‘ 的软件设计模式。

我们还指定了一个参数 :paramref:`_sa.create_engine.echo`，它将指示 :class:`_engine.Engine` 将其发出的所有 SQL 记录到一个 Python 日志记录器中，该记录器将写入标准输出。<br>
此标志是设置 :ref:`更正规的Python记录<dbengine_logging>` 的速记方式，并且对于脚本中的实验非常有用。许多 SQL 示例都会在一个名为“[SQL]”的链接下包含此 SQL 记录输出，单击该链接即可查看完整的 SQL 交互。