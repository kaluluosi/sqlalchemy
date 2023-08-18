.. highlight:: pycon+sql

.. _queryguide_toplevel:

==================
ORM 查询指南
==================

本节提供了使用 SQLAlchemy ORM 发出查询的概述，使用了  :term:`2.0 style` .
本节的读者应该熟悉   :ref:`unified_tutorial`  中的 SQLAlchemy 概述，
特别是大部分的内容都是在   :ref:`tutorial_selecting_data`  的基础上拓展的。

.. admonition:: 对于 SQLAlchemy 1.x 的用户

    在 SQLAlchemy 2.x 系列中，ORM 的 SQL SELECT 语句是使用与 Core 中相同的   :func:`_sql.select`  构造而建造的，
    然后使用   :class:`_orm.Session`  的  :meth:` _orm.Session.execute`  方法（与用于   :ref:`orm_expression_update_delete`  
    功能的   :func:`_sql.update`  和   :func:` _sql.delete`  构造相同）来调用。但是，旧版的   :class:`_query.Query` 
    对象同时执行这些步骤，作为一个“全能”的对象，它仍然是可用的，以支持在没有必要全面替换所有查询的情况下建立在 1.x 
    系列上的应用程序。关于这个对象的参考，可以查看   :ref:`query_api_toplevel` 。

.. toctree::
    :maxdepth: 3

    select
    inheritance
    dml
    columns
    relationships
    api
    query