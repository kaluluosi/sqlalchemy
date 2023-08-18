运行时检查API
===============

.. automodule:: sqlalchemy.inspection

.. autofunction:: sqlalchemy.inspect


可用的检查目标
----------------

下面列出了许多常见的检查目标。

* :class:`.Connectable`（例如:class:`_engine.Engine`，:class:`_engine.Connection`）- 返回一个:class:`_reflection`
  .Insp可进行对象检查。
* :class:`_expression.ClauseElement` - 所有SQL表达式组件，包括:class:`_schema.Table`，:class:`_schema.Column`，
  都作为它们自己的检查对象，
  这意味着将任何这些对象传递给:func:`_sa.inspect`将返回它自己。
* “对象” - 如果有一个映射，ORM将给出一个映射状态的:class:`.InstanceState`。
  :class:`.InstanceState`还提供了通过:class:`.AttributeState`接口每个属性状态的访问，以及通过:class:`.History`对象的每个刷新“历史记录”的访问。
  
  .. seealso:: 

      :ref:`orm_mapper_inspection_instancestate`

* “type”（即一个类） - ORM将针对给定的类进行映射检查 - 如果是，则返回该类的:class:`_orm.Mapper`。
  
  .. seealso::

      :ref:`orm_mapper_inspection_mapper` 

* 已映射的属性 - 将映射属性传递给:func:`_sa.inspect`，
  例如``inspect(MyClass.some_attribute)``，将返回:class:`.QueryableAttribute`对象，
  它是映射类相关联的:term:`descriptor`。这个描述符是通过它的 :attr:`.QueryableAttribute.property` 属性引用一个:class:`.MapperProperty`，
  这通常是:class:`.ColumnProperty`或 :class:`.RelationshipProperty`的实例。
* :class:`.AliasedClass` - 返回一个 :class:`.AliasedInsp` 对象。 