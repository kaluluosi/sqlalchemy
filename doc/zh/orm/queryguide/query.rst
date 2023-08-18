.. highlight:: pycon+sql
.. |prev| replace:: :doc:`api`

.. |tutorial_title| replace:: ORM查询指南

.. topic:: |tutorial_title|

      本页面是 :doc:`index` 的一部分。

      前一页: |prev|


.. currentmodule:: sqlalchemy.orm

.. _query_api_toplevel:

================
旧版查询API
================

.. admonition:: 关于旧版查询API


    本页面包含 :class:`_query.Query` 构造函数的Python生成文档，对于多年来使用SQLAlchemy ORM时唯一的SQL接口而言，这个构造函数非常重要。从版本2.0开始，一个全新的工作方式现在是标准的方法，使用ORM的同样适用于Core的 :func:`_sql.select` 构造将提供一致的查询接口。

    对于基于旧版查询API的任何应用程序来说，:class:`_query.Query` API通常代表应用程序中绝大多数数据库访问代码，因此大部分:class:`_query.Query` API **不会从SQLAlchemy 中删除**。在后台，:class:`_query.Query`对象现在在查询时将其自身转换为2.0风格的 :func:`_sql.select`对象，因此现在它只是一个非常薄的适配器API。

    对于基于:class:`_query.Query`的应用程序的迁移指南到2.0风格，请参见 :ref:`migration_20_query_usage` 。

    对于使用2.0风格为ORM对象编写SQL的概述，请从 :ref:`unified_tutorial` 开始。 2.0风格查询的其他参考内容位于 :ref:`queryguide_toplevel`。

查询对象
================

:class:`_query.Query` 是基于给定的 :class:`~.Session` 生成的，使用 :meth:`~.Session.query` 方法：

    q = session.query(SomeMappedClass)

以下是 :class:`_query.Query`对象的完整接口。

.. autoclass:: sqlalchemy.orm.Query
   :members:
   :inherited-members:

ORM特定的查询构造函数
=============================

此部分已转移到 :ref:`queryguide_additional` .