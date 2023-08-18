.. |tutorial_title| replace:: SQLAlchemy统一教程
.. |next| replace::  :doc:`engine` 

.. footer_topic:: |tutorial_title|

      下一节: |next|

.. _unified_tutorial:

.. rst-class:: orm_core

============================
SQLAlchemy统一教程
============================

.. admonition:: 关于本文档

    SQLAlchemy统一教程集成了SQLAlchemy的Core和ORM组件，是SQLAlchemy的整体介绍。对于在1.X系列中使用SQLAlchemy的2.0样式的用户，ORM使用具有Core风格的查询和_sql.select构造，Core连接和ORM会话之间的事务语义是等效的。请注意，每个部分的蓝色边框风格会告诉您一个特定主题的“ORM-ish”程度！

    已经熟悉SQLAlchemy的用户，尤其是那些想把现有应用迁移到SQLAlchemy 2.0系列的1.4过渡阶段下的用户，应该查看:migration_20_toplevel文件。

    对于初学者来说，本文档有很多细节，但到最后，他们将被视为卓越的炼金术士。

SQLAlchemy以两个不同的API呈现，一个建立在另一个之上。这些API被称为Core和ORM。

.. container:: core-header

    **SQLAlchemy Core**是SQLAlchemy作为“数据库工具包”的基础架构。该库提供了用于管理与数据库的连接、与数据库查询和结果交互、以及编程构建SQL语句的工具。

    **主要仅Core部分**将不涉及ORM。这些部分中使用的SQLAlchemy构造将从“sqlalchemy”命名空间导入。作为主题分类的另一个指示器，它们还将包括右侧的**深蓝色边框**。在使用ORM时，这些概念仍然在使用，但在用户代码中不太常见。ORM用户应该阅读这些部分，但不希望直接使用这些API进行ORM中心化代码。

.. container:: orm-header

    **SQLAlchemy ORM**基于Core提供可选的**对象关系映射**功能。ORM提供了一个附加的配置层，允许用户定义的Python类**映射**到数据库表和其他构造，以及称为**会话**的对象持久化机制。然后，它扩展了Core级别SQL表达式语言，以允许SQL查询基于用户定义的对象进行组合和调用。

    **主要仅ORM部分**应该**包含“ORM”这个短语**，以便清楚地表明这是一个ORM相关主题。这些部分中使用的SQLAlchemy构造将从“sqlalchemy.orm”命名空间导入。最后，作为主题分类的另一个指示器，它们还将包括左侧的**浅蓝色边框**。仅Core用户可以跳过这些部分。

.. container:: core-header, orm-dependency

    本教程的**大多数部分讨论的是Core概念，这些概念也在ORM中被明确使用**。特别是SQLAlchemy 2.0在ORM内具有更高程度的Core API使用集成。

    对于这些部分中的每一部分，都会有**介绍性文本**讨论ORM用户应该期望使用这些编程模式的程度。这些部分中的SQLAlchemy构造将从“sqlalchemy”命名空间导入，同时可能会使用“sqlalchemy.orm”构造。作为主题分类的另一个指示器，这些部分还将包括**左侧较薄的浅蓝色边框和右侧较厚的深蓝色边框**。Core和ORM用户应该平等地熟悉这些部分中的概念。

教程概述
==========

本教程将按应该学习的自然顺序呈现两个概念，首先是基本Core方法，然后是更多的ORM中心的概念。

本教程的主要部分如下:

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

*   :ref:`tutorial_engine`  - 所有SQLAlchemy应用程序都始于 :class:` _engine.Engine`对象；这里介绍如何创建一个。

*   :ref:`tutorial_working_with_transactions`  - 给出了 :class:` _engine.Engine`及其相关对象 :class:`_engine.Connection` 和 :class:`_result.Result` 的使用API。这一内容主要是Core-centric，但ORM用户至少也要熟悉 :class:`_result.Result` 对象。

*   :ref:`tutorial_working_with_metadata`  - SQLAlchemy的SQL抽象以及ORM依赖于将数据库架构构造定义为Python对象的系统。本节介绍如何从Core和ORM的角度做到这一点。

*   :ref:`tutorial_working_with_data`  - 在这里，我们学习如何在数据库中创建、选择、更新和删除数据。所谓的：term:` CRUD`操作在SQLAlchemy中的Core中给出，并链接到它们的ORM对应项。详细介绍了在多个教程中都介绍的SELECT操作的SD  :ref:`tutorial_selecting_data`  同样适用于 Core 和 ORM。

*   :ref:`tutorial_orm_data_manipulation`  讲述了ORM的持久性框架；基本上是ORM-centric方法来插入、更新和删除，以及如何处理事务。

*   :ref:`tutorial_orm_related_objects`  介绍了   :func:` _orm.relationship`  构造的概念，并提供了简要概述其用法的链接。

*   :ref:`tutorial_further_reading`  列出了一系列主要的顶级文档部分，充分说明了本教程中介绍的概念。

.. rst-class:: core-header, orm-dependency

版本检查
-------------

本教程使用称为“ `doctest <https://docs.python.org/3/library/doctest.html> `_”的系统编写。所有使用“ ``>>>`` ”编写的代码片段实际上都会作为SQLAlchemy的测试套件的一部分运行，并且邀请读者随时使用自己的Python解释器来使用给定的代码示例。

如果运行这些示例，则建议读者执行快速检查，以验证我们正在使用 **版本2.0** 的SQLAlchemy：

.. sourcecode:: pycon+sql

    >>> import sqlalchemy
    >>> sqlalchemy.__version__  # doctest: +SKIP
    2.0.0