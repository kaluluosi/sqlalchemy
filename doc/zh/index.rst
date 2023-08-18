:孤儿:

.. _index_toplevel:

===================================
SQLAlchemy文档
===================================

.. container:: left_right_container

  .. container:: leftmost

      .. rst-class:: h2

        入门指南

  .. container::

    初次接触SQLAlchemy吗？ 从这里开始：

    * **Python入门:** :ref:`安装指南<installation>` - 使用pip等安装基础指导

    * **Python老手:** :doc:`SQLAlchemy概述<intro>` - 简要的架构概述

.. container:: left_right_container

  .. container:: leftmost

    .. rst-class:: h2

        教程

  .. container::

    SQLAlchemy的新用户以及旧版本的Veterans应该从 :doc:`/tutorial/index` 开始，其中包含使用ORM或只使用核心时Alchemy所需了解的所有内容。

    * **快速浏览:** :doc:`/orm/quickstart` - ORM工作方式的一瞥

    * **适用于所有用户:** :doc:`/tutorial/index` - 面向核心和ORM的深入教程


.. container:: left_right_container

  .. container:: leftmost

      .. rst-class:: h2

        迁移说明

  .. container::

    来自旧版本SQLAlchemy的用户，特别是那些从1.x工作方式转换的用户，将希望查看此文档。

    * :doc:`从SQLAlchemy 1.3或1.4迁移到SQLAlchemy 2.0的完整背景 <changelog/migration_20>` 
    * :doc:`SQLAlchemy 2.0中的新功能和行为 <changelog/whatsnew_20>` - 1.x迁移之外的新功能和行为
    * :doc:`Changelog目录 <changelog/index>` - 所有SQLAlchemy版本的详细更改日志


.. container:: left_right_container

  .. container:: leftmost

      .. rst-class:: h2

      参考和操作指南


  .. container:: orm

    **SQLAlchemy ORM** - ORM的详细指南和API参考

    * **映射类:**
      :doc:`映射Python类 <orm/mapper_config>` |
      :doc:`关系配置 <orm/relationships>`

    * **ORM使用:**
      :doc:`使用ORM会话 <orm/session>` |
      :doc:`ORM查询指南 <orm/queryguide/index>` |
      :doc:`使用AsyncIO <orm/extensions/asyncio>`

    * **配置扩展:**
      :doc:`关联代理 <orm/extensions/associationproxy>` |
      :doc:`混合属性 <orm/extensions/hybrid>` |
      :doc:`可变标量 <orm/extensions/mutable>` |
      :doc:`自动映射 <orm/extensions/automap>` |
      :doc:`所有扩展 <orm/extensions/index>`

    * **扩展ORM:**
      :doc:`ORM事件和内部 <orm/extending>`

    * **其他:**
      :doc:`示例介绍 <orm/examples>`


  .. container:: core

    **SQLAlchemy核心** - 用于Core工作的详细指南和API参考

    * **引擎、连接、池:**
      :doc:`引擎配置 <core/engines>` |
      :doc:`连接，事务，结果 <core/connections>` |
      :doc:`AsyncIO支持 <orm/extensions/asyncio>` |
      :doc:`连接池 <core/pooling>`

    * **模式定义:**
      :doc:`概述<core/schema>` |
      :ref:`表和列<metadata_describing_toplevel>` |
      :ref:`数据库内省(反射) <metadata_reflection_toplevel>` |
      :ref:`插入/更新默认值<metadata_defaults_toplevel>` |
      :ref:`约束和索引<metadata_constraints_toplevel>` |
      :ref:`使用数据定义语言 (DDL) <metadata_ddl_toplevel>`

    * **SQL语句:**
      :doc:`SQL表达式元素<core/sqlelement>` |
      :doc:`操作符参考<core/operators>` |
      :doc:`SELECT和相关结构 <core/selectable>` |
      :doc:`INSERT, UPDATE, DELETE<core/dml>` |
      :doc:`SQL函数<core/functions>` |
      :doc:`目录<core/expression_api>`


    * **数据类型:**
      :ref:`概述<types_toplevel>` |
      :ref:`构建定制类型<types_custom>` |
      :ref:`类型API参考<types_api>`

    * **核心基础知识:**
      :doc:`概述<core/api_basics>` |
      :doc:`运行时内省API <core/inspection>` |
      :doc:`事件系统 <core/event>` |
      :doc:`Core事件接口<core/events>` |
      :doc:`创建自定义SQL结构<core/compiler>`

.. container:: left_right_container

    .. container:: leftmost

      .. rst-class:: h2

        方言文档

    .. container::

      方言是SQLAlchemy用于与各种类型的DBAPI和数据库通信的系统。
      本部分描述了有关各个方言的注释、选项和使用模式。

      :doc:`PostgreSQL <dialects/postgresql>` |
      :doc:`MySQL <dialects/mysql>` |
      :doc:`SQLite <dialects/sqlite>` |
      :doc:`Oracle <dialects/oracle>` |
      :doc:`Microsoft SQL Server <dialects/mssql>`

      :doc:`更多方言...<dialects/index>`

.. container:: left_right_container

  .. container:: leftmost

      .. rst-class:: h2

        补充

  .. container::

    * :doc:`常见问题解答<faq/index>` - 常见问题和解决方案收集
    * :doc:`词汇表<glossary>` - SQLAlchemy文档中使用的术语
    * :doc:`错误消息指南<errors>` - 多个SQLAlchemy错误的解释
    * :doc:`完整目录表<contents>`
    * :ref:`索引<genindex>`