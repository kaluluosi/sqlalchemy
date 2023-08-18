.. highlight:: pycon+sql

.. |prev| replace::  :doc:`metadata` 
.. |next| replace::  :doc:`data_insert` 

.. include:: tutorial_nav_include.rst

.. rst-class:: core-header, orm-addin

.. _tutorial_working_with_data:

使用数据
========

在   :ref:`tutorial_working_with_transactions`  中，我们学习了如何与 Python DBAPI 及其事务状态进行交互的基础知识。然后，在   :ref:` tutorial_working_with_metadata`  中，我们学习了如何使用   :class:`_schema.MetaData`  和相关对象在 SQLAlchemy 中表示数据库表格、列和约束。在本节中，我们将结合上述两个概念来创建、选择和操作关系数据库中的数据。我们与数据库的交互始终是基于事务的，即使我们将数据库驱动程序设置为在幕后使用   :ref:` autocommit <dbapi_autocommit>` 。

本节的组成部分如下：

*   :ref:`tutorial_core_insert`  - 为了将某些数据放入数据库中，我们引入并演示了 Core   :class:` _sql.Insert`  构造。从 ORM 的角度描述的 INSERT 将在下一节   :ref:`tutorial_orm_data_manipulation`  中描述。

*   :ref:`tutorial_selecting_data`  - 本节将详细描述   :class:` _sql.Select`  构造，这是 SQLAlchemy 中最常用的对象。  :class:`_sql.Select`  构造发射 Core 和 ORM-centric 应用程序的 SELECT 语句，两种用例都将在此描述。在稍后的   :ref:` tutorial_select_relationships`  和   :ref:`queryguide_toplevel`  中还注意到了其他 ORM 用例。

*   :ref:`tutorial_core_update_delete`  - 完成插入和数据选择之后，本节将从 Core 的角度描述   :class:` _sql.Update`  和   :class:`_sql.Delete`  构造的使用。ORM 特定的 UPDATE 和 DELETE 类似地在   :ref:` tutorial_orm_data_manipulation`  中描述。

.. toctree::
    :hidden:
    :maxdepth: 10

    data_insert
    data_select
    data_update