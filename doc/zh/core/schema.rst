.. _schema_toplevel:

==========================
Schema定义语言
==========================

.. currentmodule:: sqlalchemy.schema

本节说明SQLAlchemy的**模式元数据**，一种描述和检查数据库结构的全面系统。

SQLAlchemy查询和对象映射操作的核心由Python对象构成，这些对象描述表和其他模式级对象。这些对象是三种主要类型操作的核心——发出CREATE和DROP语句（也称为DDL），构建SQL查询和表达关于数据库中已经存在的结构的信息。

可以通过明确命名不同组件及其属性来表达数据库元数据，使用诸如:class:`~sqlalchemy.schema.Table`、:class:`~sqlalchemy.schema.Column`、:class:`~sqlalchemy.schema.ForeignKey`和:class:`~sqlalchemy.schema.Sequence`这样的结构，它们都是从``sqlalchemy.schema``包导入的。也可以通过SQLAlchemy使用称为*反射*的过程生成元数据，这意味着您从单个对象（如:class:`~sqlalchemy.schema.Table`）开始对其进行命名，然后指示SQLAlchemy从特定的引擎源中加载所有与该名称相关的其他信息。

SQLAlchemy的数据库元数据结构的一个关键特点是它们被设计成以*声明式*风格使用，其密切类似于真正的DDL。因此，它们对于那些具有创建真实模式生成脚本背景的人最直观。

.. toctree::
    :maxdepth: 3

    metadata
    reflection
    defaults
    constraints
    ddl