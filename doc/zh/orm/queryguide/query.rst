.. highlight:: pycon+sql
.. |prev| replace::  :doc:`api` 

.. |tutorial_title| replace:: ORM查询指南

.. topic:: |tutorial_title|

    本页是  :doc:`index`  的一部分。

    上一页：|prev|


.. currentmodule:: sqlalchemy.orm

.. _query_api_toplevel:

================
Legacy查询API
================

.. admonition:: 关于Legacy查询API

    本页包含Python生成的文档，针对的是 `查询（Query）` 构造函数。该构造函数多年来一直是使用SQLAlchemy ORM时的唯一SQL接口。从版本2.0开始，全新的工作方式成为标准方法，对于ORM来说，与核心库（Core）使用的相同的   :func:`_sql.select`  构造函数一样即可，为构建查询提供一致的界面。

    对于任何在2.0 API之前构建在SQLAlchemy ORM上的应用程序而言，  :class:`_query.Query`  API大多数情况下都是应用程序中的大部分数据库访问代码，因此大部分   :class:` _query.Query`  API**不会被从SQLAlchemy中删除**。当执行   :class:`_query.Query`  对象时，它在后台将自身转换为2.0样式的   :func:` _sql.select`  对象，因此它现在只是一个非常薄的适配器API。

    转换基于  :class:`_query.Query` 。

    关于2.0样式中针对ORM对象编写SQL的介绍，请从   :ref:`unified_tutorial`  开始。查询2.0样式的附加参考可参阅：  :ref:` queryguide_toplevel` 。

查询对象
=================

  :class:`_query.Query`  是在给定的   :class:` ~.Session`  中生成的，使用  :meth:`~.Session.query`  方法即可生成：

    q = session.query(SomeMappedClass)

以下是   :class:`_query.Query`  对象的完整界面。

.. autoclass:: sqlalchemy.orm.Query
   :members:
   :inherited-members:

ORM特定的查询构造函数
=============================

此部分已移至   :ref:`queryguide_additional` 。