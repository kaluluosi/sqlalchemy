.. _schema_toplevel:

========================================
模式定义语言 (Schema Definition Language)
========================================

.. currentmodule:: sqlalchemy.schema

这一节具体涉及SQLAlchemy **模式元数据**，即一套描述和检查数据库模式的全面系统。

SQLAlchemy的查询和对象映射操作的核心受到*数据库元数据*的支持，这些元数据由描述表和其他模式级对象的Python对象组成。这些对象是发出CREATE和DROP语句(称为*DDL*)、构造SQL查询以及表达有关已存在于数据库中的结构信息的三种主要类型操作的核心。

数据库的元数据可以通过明确命名各种组件及其属性来表示，使用构造如   :class:`~sqlalchemy.schema.Table` 、  :class:` ~sqlalchemy.schema.Column` 、   :class:`~sqlalchemy.schema.ForeignKey`  和   :class:` ~sqlalchemy.schema.Sequence`  的语句。它也可以通过SQLAlchemy使用一种称为*反射*的过程生成，即您从一个单一对象开始，如   :class:`~sqlalchemy.schema.Table` ，为其分配名称，然后指示SQLAlchemy从特定引擎源中加载与该名称相关的所有其他信息。

SQLAlchemy的数据库元数据构造的一个关键特点是，它们被设计成以*申明式*样式使用，这与真正的DDL非常相似。因此，它们对于那些具有创建真正的模式生成脚本的背景的人来说是最直观的。

.. toctree::
    :maxdepth: 3

    metadata
    reflection
    defaults
    constraints
    ddl