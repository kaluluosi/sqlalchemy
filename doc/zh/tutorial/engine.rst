.. |prev| replace::  :doc:`index` 
.. |next| replace::  :doc:`dbapi_transactions` 

.. include:: tutorial_nav_include.rst

.. rst-class:: core-header, orm-addin

.. _tutorial_engine:

建立连接 - Engine
======================

.. container:: orm-header

    **欢迎来到ORM和Core读者们！**

    每个连接到数据库的SQLAlchemy应用程序都需要使用一个  :class:`_engine.Engine` 。
    这个短小的部分适合每个读者。

任何SQLAlchemy应用程序的开始都是一个称为 :class:`_engine.Engine` 的对象。此对象作为连接到特定数据库的中心来源，为这些数据库连接提供工厂和一个名为  :ref:`connection pool <pooling_toplevel>` 的占用空间。通常，引擎是为特定数据库服务器创建的全局对象，使用URL字符串进行配置，描述它应该如何连接到数据库主机或后端。 

在本教程中，我们将使用一个仅存在于内存中的SQLite数据库。这是一种在不需要设置实际预先存在的数据库的情况下进行测试的简单方法。使用  :func:`_sa.create_engine` :

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)

  :class:`_sa.create_engine` ` "sqlite+pysqlite:///:memory:"`` 被传递。该字符串向 :class:`_engine.Engine` 指示三个重要事实:

1. 我们正在与哪种数据库进行通信？这是上面的 ``sqlite`` 部分，用于将SQLAlchemy连接到称为 :term:`dialect` 的对象。

2. 我们正在使用哪种  :term:`DBAPI`  ？Python  :term:` DBAPI`  是SQLAlchemy与特定数据库进行交互所使用的第三方驱动程序。在这种情况下，我们使用名称为 ``pysqlite``，在现代Python中使用的是SQLITE库的标准库接口 `sqlite3 <https://docs.python.org/library/sqlite3.html>`_。如果省略，SQLAlchemy将为所选的特定数据库使用默认的  :term:`DBAPI` 。

3. 我们如何找到数据库？在这种情况下，我们的URL包含短语 ``/  :memory:``  ，这是一个指向 ` `sqlite3`` 模块的指示符，说明我们将使用**仅存在于内存中**的数据库。这种类型的数据库非常适合进行实验，因为它不需要任何服务器，也不需要创建新文件。

.. sidebar:: 懒惰的连接

      :class:`_engine.Engine`  方法返回时，实际上还没有尝试连接到数据库；只有在第一次要求它执行针对数据库的任务时才会发生这种情况。
    这是一种称为  :term:`lazy initialization`  的软件设计模式。

我们还指定了一个参数：  :paramref:`_sa.create_engine.echo`  ，它将指示  :class:` _engine.Engine`  的一种简写方式，并且对于脚本中的实验非常有用。许多SQL示例都会在下面包括这个SQL日志输出，其中一个 ``[SQL]``链接，当单击它时，将显示完整的SQL交互。