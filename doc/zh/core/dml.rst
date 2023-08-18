插入、更新、删除
========================

INSERT、UPDATE、DELETE语句的构建从:class:`.UpdateBase`开始。:class:`_expression.Insert`和:class:`_expression.Update`构建于中间层 :class:`.ValuesBase`。

.. currentmodule:: sqlalchemy.sql.expression

.. _dml_foundational_consructors:

DML基础构造器
--------------------------------------

顶层"INSERT"、"UPDATE"、"DELETE"构造器

.. autofunction:: delete

.. autofunction:: insert

.. autofunction:: update


DML类文档构造器
--------------------------------------

:ref:`dml_foundational_consructors`列出的构造器的类文档

.. autoclass:: Delete
   :members:

   .. automethod:: Delete.where

   .. automethod:: Delete.returning

.. autoclass:: Insert
   :members:

   .. automethod:: Insert.values

   .. automethod:: Insert.returning

.. autoclass:: Update
   :members:

   .. automethod:: Update.returning

   .. automethod:: Update.where

   .. automethod:: Update.values

.. autoclass:: sqlalchemy.sql.expression.UpdateBase
   :members:

.. autoclass:: sqlalchemy.sql.expression.ValuesBase
   :members: