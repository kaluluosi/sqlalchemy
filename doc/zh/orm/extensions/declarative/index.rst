.. _declarative_toplevel:

.. currentmodule:: sqlalchemy.ext.declarative

======================
声明式扩展
======================

特定于 :ref:`Declarative <orm_declarative_mapping>` 映射API的扩展。

.. versionchanged:: 1.4  大部分的声明式扩展现在并入Sqlalchemy ORM中，
   可以从``sqlalchemy.orm``命名空间中导入。详见 :ref:`orm_declarative_mapping` 中的文档。
   关于变化的概述，请参阅 :ref:`change_5508`.

.. autoclass:: AbstractConcreteBase

.. autoclass:: ConcreteBase

.. autoclass:: DeferredReflection
   :members: