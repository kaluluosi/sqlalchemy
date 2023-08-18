类映射API
=================

.. autoclass:: registry
    :members:

.. autofunction:: add_mapped_attribute

.. autofunction:: column_property

.. autofunction:: declarative_base

.. autofunction:: declarative_mixin

.. autofunction:: as_declarative

.. autofunction:: mapped_column

.. autoclass:: declared_attr

    .. attribute:: cascading

        将:class:`.declared_attr`标记为级联，表示一个基于列或MapperProperty的声明属性在映射继承场景下应该在每个映射的子类中分别配置。

        .. warning::

            :attr:`.declared_attr.cascading` 修饰符存在以下几个限制：

            * 该标记**只**适用于声明式 mixin 类和 ``__abstract__`` 类上使用:class:`.declared_attr`。在直接映射的类上使用它目前没有效果。

            * 该标记**只**适用于通常命名的属性，如非任何特殊的下划线属性，例如 ``__tablename__``。在这些属性上它**没有**效果。

            * 该标记目前**不允许进一步的覆盖**到类层次结构中; 如果子类试图覆盖该属性，则会发出警告并跳过被覆盖的属性。这是一个现有的限制，希望在某个时间点得到解决。

        在下面的示例中，MyClass和MySubClass都将有一个独特的``id``列对象建立::

            class HasIdMixin:
                @declared_attr.cascading
                def id(cls):
                    if has_inherited_table(cls):
                        return Column(ForeignKey("myclass.id"), primary_key=True)
                    else:
                        return Column(Integer, primary_key=True)


            class MyClass(HasIdMixin, Base):
                __tablename__ = "myclass"
                # ...


            class MySubClass(MyClass):
                """ """

                # ...

        上述配置的行为是，``MySubClass``将引用它自己的``id``列以及位于``MyClass``下面``some_id``属性之上的``MyClass``的``id``列。

        .. seealso::

            :ref:`declarative_inheritance`

            :ref:`mixin_inheritance_columns`

    .. attribute:: directive

        标记:class:`.declared_attr`为修饰一个Declarative指令（如``__tablename__``或``__mapper_args__``）。

        :attr:`.declared_attr.directive`的目的是严格支持:pep:`484`类型工具，允许修饰的函数具有**不使用** :class:`_orm.Mapped`通用类的返回类型，这在对于列和映射属性使用:class:`.declared_attr`时通常是必须的。在运行时，:attr:`.declared_attr.directive`返回未经修改的:class:`.declared_attr`类。

        例如::

          class CreateTableName:
              @declared_attr.directive
              def __tablename__(cls) -> str:
                  return cls.__name__.lower()

        .. versionadded:: 2.0

        .. seealso::

            :ref:`orm_mixins_toplevel`

            :class:`_orm.declared_attr`


.. autoclass:: DeclarativeBase
    :members:
    :special-members: __table__, __mapper__, __mapper_args__, __tablename__, __table_args__

.. autoclass:: DeclarativeBaseNoMeta
    :members:
    :special-members: __table__, __mapper__, __mapper_args__, __tablename__, __table_args__

.. autofunction:: has_inherited_table

.. autofunction:: synonym_for

.. autofunction:: object_mapper

.. autofunction:: class_mapper

.. autofunction:: configure_mappers

.. autofunction:: clear_mappers

.. autofunction:: sqlalchemy.orm.util.identity_key

.. autofunction:: polymorphic_union

.. autofunction:: orm_insert_sentinel

.. autofunction:: reconstructor

.. autoclass:: Mapper
   :members:

.. autoclass:: MappedAsDataclass
    :members:

.. autoclass:: MappedClassProtocol
    :no-members: