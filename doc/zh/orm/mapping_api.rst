类映射API
=================

.. autoclass:: registry
    :members:

.. autofunction:: add_mapped_attribute
   :noindex:

.. autofunction:: column_property
   :noindex:

.. autofunction:: declarative_base
   :noindex:

.. autofunction:: declarative_mixin
   :noindex:

.. autofunction:: as_declarative
   :noindex:

.. autofunction:: mapped_column
   :noindex:

.. autoclass:: declared_attr

    .. attribute:: cascading

        标记   :class:`.declared_attr`  为层叠。

        这是一个特殊的标记，指示应针对映射的每个子类在映射继承场景中单独配置
        基于列或MapperProperty的声明的属性。

        .. warning::

            标记  :attr:`.declared_attr.cascading`  具有几个限制：

            * 该标记仅适用于在声明性混合类和 ``__abstract__`` 类上使用   :class:`.declared_attr`  ；
              当直接在映射类上使用时，它目前没有任何效果。

            * 该标记仅适用于常规命名属性，例如。不是任何特殊的下划线属性，如 ``__tablename__``。
              在这些属性上它 **没有** 效果。

            * 该标记目前 **不允许更多的覆盖** 在类层次结构下; 如果子类尝试覆盖该属性，则会发出警告并跳过覆盖的属性。
              这是一个限制，希望在某个时候得以解决。

        在下面的示例中，MyClass 和 MySubClass 都将建立一个不同的 ``id`` Column 对象::

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

        上面配置的行为是， ``MySubClass`` 将引用其自身的 ``id`` 列，以及 ``MyClass`` 在命名为 `some_id` 的属性下。

        .. seealso::

              :ref:`declarative_inheritance` 

              :ref:`mixin_inheritance_columns` 

    .. attribute:: directive

        标记   :class:`.declared_attr`  为装饰性的 Declarative 指令，例如 ` `__tablename__`` 或 ``__mapper_args__``。

         :attr:`.declared_attr.directive`  的目的严格是支持  :pep:` 484`  类型工具，
        通过允许装饰函数具有返回类型，该函数不使用   :class:`_orm.Mapped`  通用类，
        如用于列和映射属性时通常情况下会有的情况。在运行时，  :attr:`.declared_attr.directive`   返回未修改的   :class:` .declared_attr`  类。

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
   :noindex:

.. autofunction:: synonym_for
   :noindex:

.. autofunction:: object_mapper
   :noindex:

.. autofunction:: class_mapper
   :noindex:

.. autofunction:: configure_mappers
   :noindex:

.. autofunction:: clear_mappers
   :noindex:

.. autofunction:: sqlalchemy.orm.util.identity_key
   :noindex:

.. autofunction:: polymorphic_union
   :noindex:

.. autofunction:: orm_insert_sentinel
   :noindex:

.. autofunction:: reconstructor
   :noindex:

.. autoclass:: Mapper
   :members:

.. autoclass:: MappedAsDataclass
   :members: 

.. autoclass:: MappedClassProtocol
   :no-members: