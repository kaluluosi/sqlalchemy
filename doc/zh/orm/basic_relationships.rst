.. _relationship_patterns:

基本关系模式
---------------------------

这里简要介绍了基本的关系模式，并使用基于   :class:`_orm.Mapped`  注解类型的   :ref:` Declarative <orm_explicit_declarative_base>`  样式映射来说明。

以下是每个部分的配置：

    from __future__ import annotations
    from typing import List

    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass

声明式与命令式形式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

随着 SQLAlchemy 的不断发展，不同的 ORM 配置样式不断涌现。对于本节和其他使用注释的示例，应使用所需的类或字符串类名称作为传递给   :func:`_orm.relationship`  的第一个参数。下面的示例说明了本文档中使用的表格的完全声明性示例，该示例使用了  :pep:` 484`  注释，并从   :class:`_orm.Mapped`  注释派生目标类和集合类型，这是 SQLAlchemy 声明性映射的最新形式。

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
        parent: Mapped["Parent"] = relationship(back_populates="children")

相比之下，使用没有注释的 Declarative 映射是映射的“经典”形式，其中   :func:`_orm.relationship`  需要直接传递所有参数，如下面的示例所示：

    class Parent(Base):
        __tablename__ = "parent_table"

        id = mapped_column(Integer, primary_key=True)
        children = relationship("Child", back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(ForeignKey("parent_table.id"))
        parent = relationship("Parent", back_populates="children")

最后，使用   :ref:`Imperative Mapping <orm_imperative_mapping>` ，即 SQLAlchemy 在实现 Declarative 之前的映射形式（尽管仍然是一部分用户首选的映射形式），上述配置如下所示：

    registry.map_imperatively(
        Parent,
        parent_table,
        properties={"children": relationship("Child", back_populates="parent")},
    )

    registry.map_imperatively(
        Child,
        child_table,
        properties={"parent": relationship("Parent", back_populates="children")},
    )

此外，未注释映射的默认集合样式是 ``list``。要在没有注释的映射中使用 ``set`` 或其他集合，可以使用  :paramref:`_orm.relationship.collection_class`  参数指定。

    class Parent(Base):
        __tablename__ = "parent_table"

        id = mapped_column(Integer, primary_key=True)
        children = relationship("Child", collection_class=set, ...)

有关   :func:`_orm.relationship`  的集合配置的详细信息请参见   :ref:` custom_collections` 。

在需要时注意注释和未注释/命令式样式之间的其他差异。

.. _relationship_patterns_o2m:

一对多
~~~~~~~~~~~

一对多关系在子表上放置外键引用父表。在父项上指定   :func:`_orm.relationship`  时，将其指定为引用由子项表示的项目集合::

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship()


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))

为了在一对多中建立双向关系，其中“reverse”侧是多对一，可以指定一个额外的   :func:`_orm.relationship` ，并使用  :paramref:` _orm.relationship.back_populates`  参数将两个联系起来，对于每个   :func:`_orm.relationship` ，都是使用用于该另一个的  :paramref:` _orm.relationship.back_populates`  作为值属性名称。

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
        parent: Mapped["Parent"] = relationship(back_populates="children")

“Child”将获得一个多对一语义的 ``parent`` 属性。

.. _relationship_patterns_o2m_collection:

在一对多中使用集合、列表或其他集合类型
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于注释的 Declarative 映射，使用   :class:`_orm.Mapped`  中使用的集合类型派生   :func:` _orm.relationship`  的类型。可以使用``Mapped[Set["Child"]]`` 来使用 ``set`` 而不是 ``list`` 作为 ``Parent.children`` 集合的集合类型。

当使用不包括注释的形式，包括命令式映射时，可以使用  :paramref:`_orm.relationship.collection_class`  参数传递要用作集合的 Python 类。

.. seealso::

      :ref:`custom_collections`  - 包含了关于集合配置的进一步细节，包括一些将   :func:` _orm.relationship`  映射到字典的技巧。

为一对多配置删除行为
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

通常情况下，当删除所属的“父项”时，所有 “Child” 对象都应被删除。为了配置此行为，将使用在   :ref:`cascade_delete`  中描述的“删除”级联选项。还有一个额外的选项，即当将“Child”对象与其父对象断开连接时，可以删除“Child”对象本身。此行为在   :ref:` cascade_delete_orphan`  中描述。

.. seealso::

      :ref:`cascade_delete` 

      :ref:`passive_deletes` 

      :ref:`cascade_delete_orphan` 


.. _relationship_patterns_m2o:

多对一
~~~~~~~~~~~

多对一在父表中放置一个外键引用子表。在父项上声明   :func:`_orm.relationship` ，其中将创建一个新的标量保持属性::

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
        child: Mapped["Child"] = relationship()


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)

上面的示例显示了一个假定非空行为的多对一关系；下一节，  :ref:`relationship_patterns_nullable_m2o`  展示了可为空版本。

通过添加第二个   :func:`_orm.relationship`  并使用  :paramref:` _orm.relationship.back_populates`  参数在两个方向上连接两个   :func:`_orm.relationship` ，即可实现双向行为。即对于每个   :func:` _orm.relationship`  都使用另一个的  :paramref:`_orm.relationship.back_populates`  作为属性名的值。

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
        child: Mapped["Child"] = relationship(back_populates="parents")


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parents: Mapped[List["Parent"]] = relationship(back_populates="child")

.. _relationship_patterns_nullable_m2o:

可为空的多对一
^^^^^^^^^^^^^^^^^^^^

在上面的示例中，``Parent.child`` 关系未指定是否允许 ``None``；这是因为 ``Parent.child_id`` 列本身不可为空，因为它被标记为 ``Mapped[int]``。如果我们想让 ``Parent.child`` 成为可为空的多对一，可以将 ``Parent.child_id`` 和 ``Parent.child`` 都设置为 ``Optional[]``，这种情况下的配置如下所示::

    from typing import Optional


    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        child_id: Mapped[Optional[int]] = mapped_column(ForeignKey("child_table.id"))
        child: Mapped[Optional["Child"]] = relationship(back_populates="parents")


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parents: Mapped[List["Parent"]] = relationship(back_populates="child")

在上面的示例中，``Parent.child_id`` 的列将被创建为允许 ``NULL`` 值的 DDL。使用显示类型声明的   :func:`_orm.mapped_column`  时，指定 ` `child_id: Mapped[Optional[int]]`` 等效于将  :paramref:`_schema.Column.nullable`  设置为 ` `True`` on the   :class:`_schema.Column` ，而` `child_id: Mapped[int]`` 等效于将其设置为 ``False``。有关此本文的详细信息，请参见   :ref:`orm_declarative_mapped_column_nullability` 。

.. tip::

  如果使用 Python 3.10 或更高版本，则使用  :pep:`604`  语法使用` `| None`` 指示可选类型更为方便，结合  :pep:`563`  延迟注释评估使得不需要使用字符串引用类型，如下所示::

     from __future__ import annotations


     class Parent(Base):
         __tablename__ = "parent_table"

         id: Mapped[int] = mapped_column(primary_key=True)
         child_id: Mapped[int | None] = mapped_column(ForeignKey("child_table.id"))
         child: Mapped[Child | None] = relationship(back_populates="parents")


     class Child(Base):
         __tablename__ = "child_table"

         id: Mapped[int] = mapped_column(primary_key=True)
         parents: Mapped[List[Parent]] = relationship(back_populates="child")

.. _relationships_one_to_one:

一对一
~~~~~~~~~~

一对一本质上是一对多   :ref:`relationship_patterns_o2m`  关系从外键的角度来看，但表明在任何时候只有一个行引用特定的父行。

当使用注释映射和  :class:`_orm.Mapped`  注释，这将暗示 ORM 不应在任何一侧上使用集合。下面的示例说明了一种新的类 ` `Association``，它映射到名为``association`` 的   :class:`.Table` ；这个表现在现在包括了一个名叫` `extra_data`` 的额外的列，它是一个字符串值，存储在每个``Parent`` 和 ``Child`` 之间的关联中。通过将表映射到显式类，来自 ``Parent``` 到 ``Child`` 的 rudimental 访问将明确地使用 ``Association`` 方式::

    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Association(Base):
        __tablename__ = "association_table"
        left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
        right_id: Mapped[int] = mapped_column(
            ForeignKey("right_table.id"), primary_key=True
        )
        extra_data: Mapped[Optional[str]]
        child: Mapped["Child"] = relationship()


    class Parent(Base):
        __tablename__ = "left_table"
        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Association"]] = relationship()


    class Child(Base):
        __tablename__ = "right_table"
        id: Mapped[int] = mapped_column(primary_key=True)

为了示例双向版本，我们本文中添加了两个   :func:`_orm.relationship` ，并使用  :paramref:` _orm.relationship.back_populates`  将其与现有定义链接::

    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Association(Base):
        __tablename__ = "association_table"
        left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
        right_id: Mapped[int] = mapped_column(
            ForeignKey("right_table.id"), primary_key=True
        )
        extra_data: Mapped[Optional[str]]
        child: Mapped["Child"] = relationship(back_populates="parents")
        parent: Mapped["Parent"] = relationship(back_populates="children")


    class Parent(Base):
        __tablename__ = "left_table"
        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Association"]] = relationship(back_populates="parent")


    class Child(Base):
        __tablename__ = "right_table"
        id: Mapped[int] = mapped_column(primary_key=True)
        parents: Mapped[List["Association"]] = relationship(back_populates="child")

在其直接形式中使用关联对象模式需要在向父项追加子项之前将子项与关联实例相关联；类似地，从父项到子项的访问通过关联对象。

     # 创建父项，通过关联附加子项a = Association()
    a.extra_data = "some data"
    a.child = Child()
    p.children.append(a)

    #通过关联项迭代子项对象，包括关联特性
    for assoc in p.children：
        print(assoc.extra_data)
        print(assoc.child)
为了使直接访问 ``Association`` 对象是可选的，SQLAlchemy 提供了   :ref:`associationproxy_toplevel`  扩展。此扩展允许配置属性，这些属性将使用单个访问器访问两个“跳跃”，一个跳跃到关联对象，另一个跳跃到目标属性。

.. _relationships_many_to_many:

多对多
~~~~~~~~~~~~

多对多在两个类之间添加一个关联表。协会表通常是一个核心   :class:`_schema.Table`  对象或其他的 Core Selectable，例如   :class:` _sql.Join`  对象，并通过  :paramref:`_orm.relationship.secondary`  参数指示，通常，   :class:` _schema.Table`  使用与声明基类关联的   :class:`_schema.MetaData`  对象，以便   :class:` _schema.ForeignKey`  指令可以定位要链接的远程表：

    from __future__ import annotations

    from sqlalchemy import Column
    from sqlalchemy import Table
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    # note for a Core table, we use the sqlalchemy.Column construct,
    # not sqlalchemy.orm.mapped_column
    association_table = Table(
        "association_table",
        Base.metadata,
        Column("left_id", ForeignKey("left_table.id")),
        Column("right_id", ForeignKey("right_table.id")),
    )


    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List[Child]] = relationship(secondary=association_table)


    class Child(Base):
        __tablename__ = "right_table"

        id: Mapped[int] = mapped_column(primary_key=True)

为了建立双向关系，两个   :func:`_orm.relationship`  都包含集合。使用  :paramref:` _orm.relationship.back_populates`  ，并为每个微博具体指定一个公共关联表。

    from __future__ import annotations

    from sqlalchemy import Column
    from sqlalchemy import Table
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    association_table = Table(
        "association_table",
        Base.metadata,
        Column("left_id", ForeignKey("left_table.id"), primary_key=True),
        Column("right_id", ForeignKey("right_table.id"), primary_key=True),
    )


    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List[Child]] = relationship(
            secondary=association_table, back_populates="parents"
        )


    class Child(Base):
        __tablename__ = "right_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parents: Mapped[List[Parent]] = relationship(
            secondary=association_table, back_populates="children"
        )

在 Many to Many 关系的配置中，集合的配置与   :ref:`relationship_patterns_o2m`  完全相同，如在   :ref:` relationship_patterns_o2m_collection`  中所述。对于使用   :class:`_orm.Mapped`  的注释映射，可以通过在   :class:` _orm.Mapped`  通用类中使用的集合类型指示集合，例如 ``set``，来指示集合。::

    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[Set["Child"]] = relationship(secondary=association_table)

使用不包括注释的形式，包括命令式映射时，与 one-to-many 相同，可以使用  :paramref:`_orm.relationship.collection_class`  参数传递用作集合的 Python 类。

.. seealso::

      :ref:`custom_collections`  - 包含了关于集合配置的进一步细节，包括一些将   :func:` _orm.relationship`  映射到字典的技巧。

.. _relationships_many_to_many_deletion:

从 Many to Many Table 中删除行
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于  :paramref:`_orm.relationship.secondary`  参数用于   :func:` _orm.relationship`  所特有的行为，是   :class:`_schema.Table`  自动受到 INSERT 和 DELETE 语句的影响，因为将对象添加或删除到集合中。无需手动从该表中删除。删除集合中的记录将导致在 flush 时删除行：

    # 行将自动从“secondary”表中删除
    myparent.children.remove(somechild)

一个经常出现的问题是在直接传递给  :meth:`.Session.delete`  的` `Child`` 对象时，如何删除“secondary”表中的行：

    session.delete(somechild)

这里有几种可能性：

* 如果从 ``Parent`` 到 ``Child`` 存在   :func:`_orm.relationship` ，但不存在将特定` `Child`` 链接到每个 ``Parent`` 的反向关系，则 SQLAlchemy 将不会注意到在删除此特定``Child`` 对象时，它需要维护将其与``Parent ``相关联的“secondary”表。不会删除“secondary”表。* 如果存在将特定 ``Child`` 链接到每个``Parent`` 的关系，假设它称为``Child.parents``，SQLAlchemy 默认情况下会加载 ``Child.parents`` 集合以定位所有 ``Parent`` 对象，并从链接此关系的“secondary”表中删除每行。请注意，此关系不需要是双向的；SQLAlchemy 仅在严格查看与要删除的``Child`` 对象相关联的每个   :func:`_orm.relationship`  上。* 在此处使用更高效的方法是使用 ON DELETE CASCADE 指令与数据库使用的外键。假设数据库支持此功能，则可以让数据库本身在引用“child”被删除时自动删除“secondary”表中的行。使用  :paramref:` _orm.relationship.passive_deletes`  在这种情况下指示 SQLAlchemy 放弃主动加载``Child.parents`` 集合。有关此功能的详细信息，请参见   :ref:`passive_deletes` 。

请注意，这些行为仅与   :func:`_orm.relationship`  的  :paramref:` _orm.relationship.secondary`  选项相关。如果处理显式映射并且不存在于相关   :func:`_orm.relationship`  的  :paramref:` _orm.relationship.secondary`  选项中的协会表，可以使用级联规则代替以自动删除与相关实体相关的实体 - 请参阅   :ref:`unitofwork_cascades`  了解有关此功能的详细信息。

.. seealso::

      :ref:`cascade_delete_many_to_many` 

      :ref:`passive_deletes_many_to_many` 


.. _association_pattern:

关联对象
~~~~~~~~~~~~~~~~~~

关联对象模式是从多对多变体中提取出来的：当关联表包含除父表和子（或左和右）表之外的其他列时，最理想的情况是将这些列映射到自己的 ORM 映射类。映射的类会映射到在使用多对多模式时通常会指出的   :class:`.Table`  上。在关联对象模式中，不使用  :paramref:` _orm.relationship.secondary`  参数；相反，将一个类直接映射到联合表。然后使用两个单独的   :func:`_orm.relationship`  建立第一个由一个到多个的父项的链接到映射关联类，接着映射关联类到子项的多对一关系，以形成从父项到关联，再到子项的单向关联对象关系。对于双向关系，需要四个   :func:` _orm.relationship`  将关联映射类与父项和子项在两个方向上链接起来。

下面的示例说明了一个新类``Association``，它映射到名为 ``association`` 的   :class:`.Table` ；现在，这个表包括一个名为` `extra_data`` 的附加列，它是一个字符串值，与``Parent`` 和``Child`` 之间的每个关联一起存储。将表映射到显式类，也可以从``Parent`` 到``Child`` 的访问，明确使用``Association``：：

    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Association(Base):
        __tablename__ = "association_table"
        left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
        right_id: Mapped[int] = mapped_column(
            ForeignKey("right_table.id"), primary_key=True
        )
        extra_data: Mapped[Optional[str]]
        child: Mapped["Child"] = relationship()


    class Parent(Base):
        __tablename__ = "left_table"
        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Association"]] = relationship()


    class Child(Base):
        __tablename__ = "right_table"
        id: Mapped[int] = mapped_column(primary_key=True)

为了说明双向版本，我们需要使用  :paramref:`_orm.relationship.back_populates` ，并使用现有的定义将 Grouped_Association_Class 链接到它们当中的每个父项和子项。::

    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Association(Base):
        __tablename__ = "association_table"
        left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
        right_id: Mapped[int] = mapped_column(
            ForeignKey("right_table.id"), primary_key=True
        )
        extra_data: Mapped[Optional[str]]
        child: Mapped["Child"] = relationship(back_populates="parents")
        parent: Mapped["Parent"] = relationship(back_populates="children")


    class Parent(Base):
        __tablename__ = "left_table"
        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Association"]] = relationship(back_populates="parent")


    class Child(Base):
        __tablename__ = "right_table"
        id: Mapped[int] = mapped_column(primary_key=True)
        parents: Mapped[List["Association"]] = relationship(back_populates="child")

在关联模式下的工作，要求必须在附加到父项之前将子项与关联实例相关联，反之亦然；类似地，从父项到子项的访问将通过关联对象。

    #创建操作,通过关联附加子项。
    p = Parent()
    a = Association(extra_data="some data")
    a.child = Child()
    p.children.append(a)

    # 通过关联项迭代子项对象,包括关联属性
    for assoc in p.children:
        print(assoc.extra_data)
        print(assoc.child)

要增强关联对象模式，以使直接访问 ``Association`` 对象是可选的，SQLAlchemy 提供了   :ref:`associationproxy_toplevel`  扩展。此扩展允许配置属性，这些属性将使用单个访问器访问两个“跳跃”，一个跳跃到关联对象，另一个跳跃到目标属性。.. seealso::

      :ref:`associationproxy_toplevel`  - 允许一种直接的“多对多”样式
    的访问方式，通过父对象和子对象关联实现三类关联对象映射。

.. warning::

    避免将关联对象模式与   :ref:`many-to-many <relationships_many_to_many>`  模式直接混合使用，
    因为这会导致数据读取和写入不一致（除非采取特殊措施）；
      :ref:`association proxy <associationproxy_toplevel>`  通常用于提供更简洁的访问。
    关于此组合带来的注意事项的更详细背景，请参见下一节   :ref:`association_pattern_w_m2m` 。

.. _association_pattern_w_m2m:

将关联对象与多对多访问模式结合使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如前一节所述，关联对象模式不会自动集成到多对多模式上。
由此产生的结果是，读操作可能会返回冲突的数据，写操作也可能会尝试刷新冲突的更改，
导致完整性错误或意外的插入或删除操作。

为了说明这一点，下面的示例通过 ``Parent.children`` 和 ``Child.parents`` 之间的双向多对多关系
以及 ``Parent.child_associations`` 到 ``Association.child`` 和 ``Child.parent_associations``
之间的关联对象关系来配置模型映射：

    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Association(Base):
        __tablename__ = "association_table"

        left_id: Mapped[int] = mapped_column(ForeignKey("left_table.id"), primary_key=True)
        right_id: Mapped[int] = mapped_column(
            ForeignKey("right_table.id"), primary_key=True
        )
        extra_data: Mapped[Optional[str]]

        # Association -> Child 之间的关联关系
        child: Mapped["Child"] = relationship(back_populates="parent_associations")

        # Association -> Parent 之间的关联关系
        parent: Mapped["Parent"] = relationship(back_populates="child_associations")


    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)

        # 到Child 的多对多关系，绕过了 `Association` 类
        children: Mapped[List["Child"]] = relationship(
            secondary="association_table", back_populates="parents"
        )

        # Parent -> Association -> Child 之间的关联关系
        child_associations: Mapped[List["Association"]] = relationship(
            back_populates="parent"
        )


    class Child(Base):
        __tablename__ = "right_table"

        id: Mapped[int] = mapped_column(primary_key=True)

        # 到Parent 的多对多关系，绕过了 `Association` 类
        parents: Mapped[List["Parent"]] = relationship(
            secondary="association_table", back_populates="children"
        )

        # Child -> Association -> Parent 之间的关联关系
        parent_associations: Mapped[List["Association"]] = relationship(
            back_populates="child"
        )

在使用此 ORM 模型进行更改操作时，对 ``Parent.children`` 进行的更改不会与 Python 中对 ``Parent.child_associations``
或 ``Child.parent_associations`` 进行的更改进行协调；
虽然所有这些关系将继续正常运行，但是对一个的更改将不会出现在另一个中，
直到   :class:`.Session`  过期，这通常在  :meth:` .Session.commit`  后自动发生。

此外，如果进行冲突的更改，例如同时添加一个新的 ``Association`` 对象，
同时将相同的关联 ``Child`` 添加到 ``Parent.children`` 中，那么当工作单元的刷新过程进行时，
例如以下示例，将会导致完整性错误：

      p1 = Parent()
      c1 = Child()
      p1.children.append(c1)

      # 冗余，将在 Association 上引发重复的 INSERT
      p1.child_associations.append(Association(child=c1))

直接将 ``Child`` 添加到 ``Parent.children`` 还意味着在 ``association`` 表中创建行，而不指示
``association.extra_data`` 列的任何值，其将接收其值为 ``NULL`` 的值。

如果知道自己在做什么，就可以使用上述映射；在以下情况下，
可能有充分理由使用多对多关系，即在使用“关联对象”模式不频繁的情况下，
在单个多对多关系中加载关系更容易，这也可以优化使用 SQL 语句中的“secondary”表稍微好一些，
而不是使用两个到显式关联类的单独关系。即使在经过谨慎组织以避免过时读取的代码访问多对多集合的情况下，
使用关联对象关系也可能是可行的，甚至可以在极端情况下直接使用  :meth:`_orm.Session.expire` 
来导致集合在当前事务中被刷新。上述模式的一个受欢迎的替代方案是使用直接的多对多
``Parent.children`` 和 ``Child.parents`` 关系的扩展，该扩展将透明地代理
通过“Association”类，同时从 ORM 的角度保持一切一致。此扩展称为   :ref:`Association Proxy <associationproxy_toplevel>` 。

.. seealso::

      :ref:`associationproxy_toplevel`  - 允许一种直接的“多对多”样式
    的访问方式，通过父对象和子对象关联实现三类关联对象映射。

.. _orm_declarative_relationship_eval:

推迟计算关系参数
~~~~~~~~~~~~~~~~~~~

前面几节中的大多数示例说明了映射如何使用字符串名称引用其目标类，
而不是实际的类本身，例如在使用 ：class:`_orm.Mapped` 时，将生成一个前向引用，
它仅在运行时作为字符串存在：

    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(back_populates="parent")


    class Child(Base):
        # ...

        parent: Mapped["Parent"] = relationship(back_populates="children")

同样，当使用非注释形式时，如非注释性 Declarative 或 Imperative 映射，
  :func:`_orm.relationship`  结构也支持字符串名称直接传递给主类参数：

    registry.map_imperatively(
        Parent,
        parent_table,
        properties={"children": relationship("Child", back_populates="parent")},
    )

对于除了主要参数之外的其他参数，如果它们依赖于尚未定义的类上的列，
则也可以指定字符串，这些字符串可能作为 Python 函数或更常见的是字符串进行调用。
对于这些参数中除主要参数外的大多数参数，除主要参数外的字符串输入
与   :func:`eval()`  函数一起用于 **以 Python 表达式的形式进行评估**，
因为它们意图接收完整的 SQL 表达式。

.. warning::

    由于 Python 的 ``eval()`` 函数用于解释传递给   :func:`_orm.relationship`  的
    对字符串的评估，因此这些参数 **不应从事接收不可信用户输入的重用**；
    ``eval()`` 对不受信任的用户输入 **不安全**。

可以在此评估中使用的完整命名空间包括为此声明基本映射的所有类，
以及 ``sqlalchemy`` 包的内容，包括表达式函数，例如   :func:`_sql.desc`  和  :attr:` _functions.func` ：

    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(
            order_by="desc(Child.email_address)",
            primaryjoin="Parent.id == Child.parent_id",
        )

在存在多个模块中具有相同名称的类的情况下，字符串类名称也可以在其中任何一个字符串表达式内
指定为模块限定路径：

    class Parent(Base):
        # ...

        children: Mapped[List["myapp.mymodel.Child"]] = relationship(
            order_by="desc(myapp.mymodel.Child.email_address)",
            primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
        )

在上面的示例中，可以通过直接将类位置字符串传递到  :paramref:`_orm.relationship.argument` 
中来通过名称消除二义性。下面展示了对 ``Child`` 的类型仅为导入的示例，与
将运行时指示符与查找正确名称的目标类相结合。

    import typing

    if typing.TYPE_CHECKING:
        from myapp.mymodel import Child


    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(
            "myapp.mymodel.Child",
            order_by="desc(myapp.mymodel.Child.email_address)",
            primaryjoin="myapp.mymodel.Parent.id == myapp.mymodel.Child.parent_id",
        )

限定路径可以是任何消除名称之间歧义的部分路径。例如，为了消除
``myapp.model1.Child`` 和 ``myapp.model2.Child`` 之间的二义性，
我们可以指定 ``model1.Child`` 或 ``model2.Child``：

    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(
            "model1.Child",
            order_by="desc(mymodel1.Child.email_address)",
            primaryjoin="Parent.id == model1.Child.parent_id",
        )

  :func:`_orm.relationship`  结构也接受 Python 函数或 lambda 表达式作为这些参数的输入。
Python 函数的方法可能如下所示：

    import typing

    from sqlalchemy import desc

    if typing.TYPE_CHECKING:
        from myapplication import Child


    def _resolve_child_model():
        from myapplication import Child

        return Child


    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(
            _resolve_child_model,
            order_by=lambda: desc(_resolve_child_model().email_address),
            primaryjoin=lambda: Parent.id == _resolve_child_model().parent_id,
        )

可以将 Python 函数或 lambda 表达式作为这些参数的输入的完整列表是：

*  :paramref:`_orm.relationship.order_by` 

*  :paramref:`_orm.relationship.primaryjoin` 

*  :paramref:`_orm.relationship.secondaryjoin` 

*  :paramref:`_orm.relationship.secondary` 

*  :paramref:`_orm.relationship.remote_side` 

*  :paramref:`_orm.relationship.foreign_keys` 

*  :paramref:`_orm.relationship._user_defined_foreign_keys` 

.. warning::

    正如前面所述，属性中将这些参数传递给   :func:`_orm.relationship` 
    **使用 eval() 函数将其作为 Python 代码表达式进行评估。不要将未经检查的输入传递给这些参数。**

在声明之后将关系添加到映射类
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

需要注意的是，在与   :ref:`orm_declarative_table_adding_columns`  中描述的方式类似，
可以在任何时候将   :class:`_orm.MapperProperty`  结构添加到声明式基础映射中，
但是：class:`_orm.Mapped` 注释类型在这种情况下不支持；因此，必须直接在   :func:`_orm.relationship` 
构造中将相关类指定为类本身、类名称的字符串或返回指向目标类的引用的可调用函数。

.. note:: 与 ORM 映射列一样，如果映射了类且要将映射化的列分配给已映射的类，
    则只有在使用“声明性基础”类时将可以正常工作，
    表示用户定义的   :class:`_orm.DeclarativeBase`  子类或由   :func:` _orm.declarative_base` 
    或  :meth:`_orm.registry.generate_base`  返回的动态生成的类。
    此“基础”类包括 Python 元类，该元类实现了一个特殊的 ``__setattr__()`` 方法，
    该方法截取了这些操作。

    如果使用像  :meth:`_orm.registry.mapped`  或类似的声明性映射或命令式函数
    （例如  :meth:`_orm.registry.map_imperatively` ）中的装饰器映射映射了类，
    则无法对映射的类进行运行时属性分配。



.. _orm_declarative_relationship_secondary_eval:

使用延迟评估形式的多对多关系的“secondary”参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

多对多关系使用  :paramref:`_orm.relationship.secondary`  参数，
该参数通常指向一个通常非映射的   :class:`_schema.Table`  对象或其他 Core 可选择对象。
支持使用 lambda 可调用或字符串名称进行延迟评估，
其中字符串解析通过使用 Python 表达式将标识符名称链接到同名   :class:`_schema.Table`  对象来完成，
这些对象存在于当前经向某个   :class:`_orm.registry`  映射的   :class:` _schema.MetaData`  集合中。

例如，在   :ref:`relationships_many_to_many`  中给出的示例中，如果我们假设 ` `association_table``
  :class:`.Table`  对象定义在映射类本身之后的一点处，我们可以使用 lambda 写入   :func:` _orm.relationship` ，
例如：

    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(
            "Child", secondary=lambda: association_table
        )

或者，以名称定位相同的   :class:`.Table`  对象，然后将名称作为参数使用。
从 Python 的角度讲，这是一个字符串表达式，在此表达式的 Python 环境中，标识符名称链接到
当前   :class:`_orm.registry`  指向的   :class:` _schema.MetaData`  集合中的表的名称：

    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(secondary="association_table")

.. warning:: 当作为字符串传递时，
     :paramref:`_orm.relationship.secondary`  参数使用 Python 的 ` `eval()`` 函数进行解析，
    即使该字符串通常是表的名称。
    **不要将不可信用户输入传递给此字符串**。