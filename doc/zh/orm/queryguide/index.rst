.. highlight:: pycon+sql

.. _ORM查询指南:

==================
ORM查询指南
==================

本节概述使用SQLAlchemy ORM使用:term:`2.0 style`语法来发出查询。

本节的读者应该熟悉:ref:`unified_tutorial`中的SQLAlchemy概述，特别是本节的大部分内容是在扩展:ref:`tutorial_selecting_data`中的内容。

.. admonition:: 对于SQLAlchemy 1.x的用户

   在SQLAlchemy 2.x系列中，ORM的SQL SELECT语句是使用与Core相同的:func:`_sql.select`构造构建的，然后在:class:`_orm.Session`中使用:meth:`_orm.Session.execute`方法调用（与现在用于:ref:`orm_expression_update_delete`特性的:func:`_sql.update`和:func:`_sql.delete`构造相同）。然而，旧版的:class:`_query.Query`对象在这个新系统上执行这些相同的步骤，作为更多"一体化"对象的一种，仍然保持可用，以支持在1.x系列上构建的应用程序，而无需全面替换所有的查询。有关此对象的参考，请参阅:ref:`query_api_toplevel`章节。




.. toctree::
    :maxdepth: 3

    select
    inheritance
    dml
    columns
    relationships
    api
    query