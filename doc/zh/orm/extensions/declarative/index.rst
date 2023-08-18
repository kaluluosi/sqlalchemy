.. _declarative_toplevel:

.. currentmodule:: sqlalchemy.ext.declarative

=====================================
声明式扩展
=====================================

该模块提供了与  :ref:`Declarative <orm_declarative_mapping>`  ORM 映射 API 特定的扩展。

.. versionchanged:: 1.4  现在大部分声明式扩展已经融入了 SQLAlchemy ORM 并可以从 ``sqlalchemy.orm`` 命名空间中导入。
   请查看   :ref:`orm_declarative_mapping`  文档以获取新的文档说明。
   针对这次变更的详细介绍请参见   :ref:`change_5508` 。

.. autoclass:: AbstractConcreteBase

.. autoclass:: ConcreteBase

.. autoclass:: DeferredReflection
   :members: