插入、更新、删除
========================

INSERT、UPDATE 和 DELETE语句基于   :class:`.UpdateBase`  开始的层次结构。  :class:` _expression.Insert`  和   :class:`_expression.Update`  结构以中介者   :class:` .ValuesBase`  为基础。

.. currentmodule:: sqlalchemy.sql.expression

.. _dml_foundational_consructors:

DML 基础构造
--------------------------------------

顶级的 "INSERT"、"UPDATE" 和 "DELETE" 构造器。

.. autofunction:: delete

.. autofunction:: insert

.. autofunction:: update


DML 类构造函数文档
--------------------------------------

构造器列表的类文档   :ref:`dml_foundational_consructors`  。

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