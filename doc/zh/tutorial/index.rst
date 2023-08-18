.. |tutorial_title| replace:: SQLAlchemy统一教程
.. |next| replace:: :doc:`engine`

.. footer_topic:: |tutorial_title|

      下一节: |next|

.. _unified_tutorial:

.. rst-class:: orm_core

============================
SQLAlchemy统一教程
============================

.. admonition:: 关于本文档

    SQLAlchemy统一教程是SQLAlchemy核心和ORM组件之间的集成，并作为对SQLAlchemy的整体介绍。对于在1.x系列中使用SQLAlchemy的用户，在工作方式上采用2.0风格，ORM使用基于Core的查询和_sql.select构造，Core连接和ORM会话之间的事务语义是等价的。请注意，每个部分的蓝色边框风格，这将告诉您特定主题的ORM是否很“ORMish”！

    熟悉SQLAlchemy的用户，尤其是那些想要在1.4过渡期内将现有应用程序移植到SQLAlchemy 2.0系列下的用户，应该同样查看迁移20_toplevel文档。

    对于新手，本文档有很多细节，但最终他们将被视为铸金术士。

SQLAlchemy被呈现为两个不同的API，一个建立在另一个之上。这些API称为**Core**和**ORM**。

.. container:: core-header

    **SQLAlchemy Core**是SQLAlchemy作为“数据库工具包”的基础架构。该库提供了管理与数据库的连接性、与数据库查询和结果的交互、以及SQL语句的编程构造的工具。

    主要是Core类型的部分将不涉及ORM。这些部分中使用的SQLAlchemy构造将从“sqlalchemy”命名空间导入。作为主题分类的附加指标，它们还将在右边包括**深蓝色边框**。当使用ORM时，这些概念仍然有效，但在用户代码中不太常见。ORM用户应阅读这些文章，但不要指望为ORM中心的代码直接使用这些API。

.. container:: orm-header

    **SQLAlchemy ORM**建立在Core之上，提供了可选的**对象关系映射**功能。ORM提供了一个额外的配置层，允许用户定义的Python类被**映射**到数据库表和其他构造，以及称为**Session**的对象持久性机制。然后，它扩展了Core级别的SQL表达式语言，允许SQL查询以用户定义的对象的方式组合和调用。

    主要是ORM类型的部分应**命名中包括“ORM”**，以便清楚这是与ORM相关的主题。这些部分中使用的SQLAlchemy构造将从“sqlalchemy.orm”名称空间导入。最后，作为附加的主题分类指标，它们还将在左边包括**淡蓝色边框**。仅使用Core的用户可以跳过这些部分。

.. container:: core-header, orm-dependency

    **大多数**章节在本教程中讨论了**Core概念，也有明确在ORM中使用**。特别是SQLAlchemy 2.0具有更高级别的Core API在ORM中的整合程度。

    对于这些部分，将有**介绍性文本**，讨论ORM用户应该期望使用这些编程模式的程度。在这些部分中，SQLAlchemy构造将从“sqlalchemy”命名空间导入，并且有可能同时使用“sqlalchemy.orm”构造。作为主题分类的附加指标，这些部分还将包括**左侧较薄的淡色边框和右侧较厚的深色边框**。Core和ORM用户应该同样熟悉这些部分中的概念。

教程概述
=================

教程将按照自然顺序呈现两个概念，首先是基本核心方法，然后扩展到更多ORM概念。

本教程的主要部分如下：

.. toctree::
    :hidden:
    :maxdepth: 10

    engine
    dbapi_transactions
    metadata
    data
    orm_data_manipulation
    orm_related_objects
    further_reading

* :ref:`tutorial_engine` - 所有SQLAlchemy应用程序都从一个:class:`_engine.Engine`对象开始；这里是如何创建的。

* :ref:`tutorial_working_with_transactions` - 介绍了类:_engine.Engine及其相关对象:class:`_engine.Connection`和:class:`_result.Result`的使用API。此内容侧重于Core，但ORM用户至少需要熟悉:class:`_result.Result`对象。

* :ref:`tutorial_working_with_metadata` - SQLAlchemy的SQL抽象以及ORM依赖于将数据库模式构件定义为Python对象的系统。本部分介绍了如何从Core和ORM的角度进行定义。

* :ref:`tutorial_working_with_data` - 在这里，我们学习如何在数据库中创建、选择、更新和删除数据。这些所谓的:term:`CRUD`(创建、读取、更新和删除)操作是以SQLAlchemy Core形式给出的，同时还有指向它们的ORM对应操作的链接。在详细介绍“:ref:`tutorial_selecting_data`”的SELECT操作同样适用于Core和ORM。

* :ref:`tutorial_orm_data_manipulation`涵盖了ORM的持久化框架。基本上是ORM中心的插入、更新和删除方式，以及如何处理事务。

* :ref:`tutorial_orm_related_objects`介绍了:func:`_orm.relationship`构造的概念，并提供了简要概述及其使用方法以及深入文档的链接。

* :ref:`tutorial_further_reading`列出了一系列完全记录本教程中介绍概念的主要顶级文档部分。


.. rst-class:: core-header, orm-dependency

版本检查
-------------

本教程是使用称为 `doctest<https://docs.python.org/3/library/doctest.html>`_ 的系统编写的。所有带有“>>>”的代码段实际上是作为SQLAlchemy的测试套件的一部分运行的，读者被邀请实时使用自己的Python解释器处理给定的代码示例。

如果运行示例，则建议读者执行快速检查以验证我们是否在**2.0版本**的SQLAlchemy上：

.. sourcecode:: pycon+sql

    >>> import sqlalchemy
    >>> sqlalchemy.__version__  # doctest: +SKIP
    2.0.0