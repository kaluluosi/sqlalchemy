.. highlight:: pycon+sql

.. |prev| replace:: :doc:`metadata`
.. |next| replace:: :doc:`data_insert`

.. include:: tutorial_nav_include.rst

.. rst-class:: core-header, orm-addin

.. _tutorial_working_with_data:

处理数据
==================

在 :ref:`tutorial_working_with_transactions` 这一部分中，我们学习了如何与Python DBAPI以及其事务状态进行交互的基础知识。 然后，在: ref:`tutorial_working_with_metadata`中，我们学习了如何使用`_schema.MetaData`和相关对象在SQLAlchemy中表示数据库表、列和约束。在本节中，我们将结合上述两个概念，创建、选择和操作关系数据库中的数据。即使我们将数据库驱动程序设置为在幕后使用: ref:`autocommit<dbapi_autocommit>`，我们与数据库的交互也始终是基于事务的。

本节的组成部分如下:

* :ref:`tutorial_core_insert` - 要将一些数据插入数据库，我们介绍并演示了Core :class:`_sql.Insert`构造。 插入是从ORM角度描述在下一节 :ref:`tutorial_orm_data_manipulation`中。

* :ref:`tutorial_selecting_data` - 本节将详细介绍:class:`_sql.Select`构造，这是SQLAlchemy中最常用的对象。 :class:`_sql.Select`构造发出Core和ORM-centric应用程序的SELECT语句，并将两种用例描述在本文中。稍后的 :ref:`tutorial_select_relationships`部分以及 :ref:`queryguide_toplevel`中还注意到了其他ORM用例。

* :ref:`tutorial_core_update_delete` - 围绕插入和选择数据，本节将从Core的角度描述:class:`_sql.Update`和:class:`_sql.Delete`构造的用法。 ORM特定的UPDATE和DELETE在 `tutorial_orm_data_manipulation` 部分中同样有所描述。

.. toctree::
    :hidden:
    :maxdepth: 10

    data_insert
    data_select
    data_update