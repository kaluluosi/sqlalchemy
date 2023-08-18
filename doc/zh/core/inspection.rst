运行时检查API
==============

.. automodule:: sqlalchemy.inspection

.. autofunction:: sqlalchemy.inspect


可用的检查目标
----------------

下面是许多常见检查目标的列表。

*   :class:`.Connectable` （即  :class:` _engine.Engine` ，  :class:`_engine.Connection` ） - 返回一个   :class:` _reflection.Inspector`  对象。
*   :class:`_expression.ClauseElement`  - 所有SQL表达式组件，包括   :class:` _schema.Table` ，  :class:`_schema.Column` ，作为它们自己的检查对象，
  意味着传递给   :func:`_sa.inspect`  的任何这些对象都将返回它们自己。
* ``object`` - 给定的对象将通过ORM进行映射检查 -
  如果是这样，将返回一个表示对象映射状态的   :class:`.InstanceState` 。
    :class:`.InstanceState` .AttributeState` 接口提供每个属性状态的访问，
  以及通过   :class:`.History`  对象提供每个刷新的属性的“历史记录”访问。

  .. seealso::

        :ref:`orm_mapper_inspection_instancestate` 

* ``类型``（即类） - 给定的类将通过ORM进行映射检查 - 如果是这样，将返回该类的   :class:`_orm.Mapper` 。

  .. seealso::

        :ref:`orm_mapper_inspection_mapper` 

* 映射属性 - 将映射的属性传递给   :func:`_sa.inspect` ，例如 ` `inspect(MyClass.some_attribute)`` 返回一个   :class:`.QueryableAttribute`  对象，
  它是与映射类相关联的  :term:`描述符` 。该描述符是通过它的  :attr:` .QueryableAttribute.property`  属性
  引用   :class:`.MapperProperty` ，通常是   :class:` .ColumnProperty`  或   :class:`.RelationshipProperty`  的一个实例。
*   :class:`.AliasedClass`  - 返回一个   :class:` .AliasedInsp`  对象。