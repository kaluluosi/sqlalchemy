.. _relationship_patterns:

关系模式基础知识
---------------------------

本节将快速讲解基本的关系模式，使用 :ref:`Declarative <orm_explicit_declarative_base>` 样式映射，这些映射基于 :class:`_orm.Mapped` 注解类型。

以下是每个部分的设置::

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

声明式 vs. 命令式形式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

随着 SQLAlchemy 的演化，不同的 ORM 配置样式已经出现过。在本节的示例代码以及其他使用 :ref:`Declarative <orm_explicit_declarative_base>` 映射并带有 :class:`_orm.Mapped` 注解的代码中，对应的非注解形式应该使用所需的类或字符串类名作为传递给 :func:`_orm.relationship` 的第一个参数。下面的示例显示了本文档中使用的形式，这是一个完全使用 :pep:`484` 注释的 Declarative 示例，其中 :func:`_orm.relationship` 结构也通过从 :class:`_orm.Mapped` 注解中派生目标类和集合类型来推导，这是 SQLAlchemy Declarative 映射的最新形式::

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
        parent: Mapped["Parent"] = relationship(back_populates="children")

相比之下，使用不带注解的 Declarative 映射是映射的“经典”形式，其中 :func:`_orm.relationship` 要求直接传递所有参数，如下面的示例所示::

    class Parent(Base):
        __tablename__ = "parent_table"

        id = mapped_column(Integer, primary_key=True)
        children = relationship("Child", back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(ForeignKey("parent_table.id"))
        parent = relationship("Parent", back_populates="children")

最后，使用 :ref:`命令式映射 <orm_imperative_mapping>`，这是 SQLAlchemy 的 Declarative 映射之前的原始映射形式（尽管它仍然是少数用户中的首选），以上面的配置为例子，其形式如下所示::

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

还可以通过 :paramref:`_orm.relationship.collection_class` 参数指定非注解映射的默认集合类型为 ``list``。若要在不使用注解的情况下使用 ``set`` 或其他集合，需要使用该参数指定。

.. _relationship_patterns_o2m:

一对多
~~~~~~~~~~~

一对多关系在子表上放置一个外键，引用父表。然后，在父表上使用 :func:`_orm.relationship` 指定该关系，引用一个由子项表示的项目集合::

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship()


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))

在一对多关系中建立双向关系（其中“反向”方向是多对一），要指定一个额外的 :func:`_orm.relationship` 并使用 :paramref:`_orm.relationship.back_populates` 参数连接两个关系，分别使用每个关系的属性名称作为 :paramref:`_orm.relationship.back_populates` 值即可：

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
        parent: Mapped["Parent"] = relationship(back_populates="children")

``Child`` 将得到一个具有多对一语义的 ``parent`` 属性。

.. _relationship_patterns_o2m_collection:

对一使用集合、列表或其他集合类型
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用带注解的 Declarative 映射，可以从 :class:`_orm.Mapped` 容器类型中使用的集合类型推断出用于 :func:`_orm.relationship` 的集合类型。下面的示例可以使用 ``set`` 而不是 ``list``，因为 ``Parent.children`` 集合可以使用 ``Mapped[Set["Child"]]``：

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[Set["Child"]] = relationship(back_populates="parent")

而在不使用注解的 Declarative 形式中包括命令式映射时，Python 类可通过使用 :paramref:`_orm.relationship.collection_class` 参数传递以指定要用作集合的类型。

.. seealso::

    :ref:`custom_collections` - 包含有关集合配置的进一步细节，包括一些将 :func:`_orm.relationship` 映射到字典的技术。


为一对多配置删除行为
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

通常情况下，当删除其所属的“Parent”时，所有“Child”对象都应该被删除。要配置此行为，使用在 :ref:`cascade_delete` 中描述的“delete”级联选项。另一个选项是，当从其父对象中取消关联时，也可以自行删除“Child”对象。此行为在 :ref:`cascade_delete_orphan` 中描述。

.. seealso::

    :ref:`cascade_delete`

    :ref:`passive_deletes`

    :ref:`cascade_delete_orphan`


.. _relationship_patterns_m2o:

多对一
~~~~~~~~~~~

多对一在父表中放置一个外键，引用子表。在父表上声明 :func:`_orm.relationship`，其中将创建一个新的标量保持属性::

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        child_id: Mapped[int] = mapped_column(ForeignKey("child_table.id"))
        child: Mapped["Child"] = relationship()


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)

以上示例显示了一个多对一关系，假设是非空行为；下一部分 :ref:`relationship_patterns_nullable_m2o` 将说明一个可空版本。

通过添加第二个 :func:`_orm.relationship` 并在两个关系中使用 :paramref:`_orm.relationship.back_populates` 参数即可实现双向关系，使用每个 :func:`_orm.relationship` 的属性名称作为 :paramref:`_orm.relationship.back_populates` 上另一个的值：


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

可空的多对一
^^^^^^^^^^^^^^^^

在上面的示例中，``Parent.child`` 关系的类型不是允许 ``None``；这来自于 ``Parent.child_id`` 本身不是可空的，因为它使用了 ``Mapped[int]`` 类型。如果希望``Parent.child`` 成为 **可空的** 多对一，我们可以将 ``Parent.child_id`` 和 ``Parent.child`` 都设置为 ``Optional[]``，在这种情况下，配置将如下所示：

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

以上，``Parent.child_id`` 的列将在 DDL 中创建为允许 ``NULL`` 值。在使用 :func:`_orm.mapped_column` 和显式类型声明时，指定 ``child_id: Mapped[Optional[int]]`` 等价于在 :class:`_schema.Column` 上设置 :paramref:`_schema.Column.nullable` 为 ``True``，而``child_id: Mapped[int]`` 等价于将其设置为``False``。请参阅 :ref:`orm_declarative_mapped_column_nullability` 以了解此行为的背景。

.. tip::

  如果使用的是 Python 3.10 或更高版本，使用 ``| None`` 指示可选类型的 :pep:`604` 语法更方便，这样在与 :pep:`563` 延迟注释评估相结合时，就不需要使用字符串引用类型：

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

从外键的角度来看，一对一本质上是一个 :ref:`relationship_patterns_o2m` 关系，但是表明在任何时候只有一行引用特定父行的情况下。使用带注解的映射 :class:`_orm.Mapped`，使用非集合类型，在关系两端上同时使用即可实现“一对一”约定。非注解映射需要显式使用 :paramref:`_orm.relationship.uselist` 参数设置为 ``False``，如下面的示例所示：

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        child: Mapped["Child"] = relationship(back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent_table.id"))
        parent: Mapped["Parent"] = relationship(back_populates="child")

以上，在加载 ``Parent`` 对象时，``Parent.child`` 属性将指向单个 ``Child`` 对象而不是集合。如果将 ``Parent.child`` 的值替换为一个新的 ``Child`` 对象，则 ORM 的工作单元过程将使用新值替换以前的 ``Child`` 行，并在默认情况下将先前的 ``child.parent_id`` 列设置为 NULL，除非设置了指定的 :ref:`cascade <unitofwork_cascades>` 行为。

.. tip::

  如前所述，ORM 将“一对一”模式视为约定，当它在 ``Parent`` 对象上加载 ``Parent.child`` 属性时，假定仅会返回一行。如果返回多行，ORM 将发出警告。

  然而，上述关系的 ``Child.parent`` 仍然保持为“多对一”，并且 ORM 本身没有内在机制来防止在持久化期间针对同一``Parent`` 创建多个``Child`` 对象。相反，可以在实际数据库架构中使用 :ref:`唯一约束 <schema_unique_constraint>` 等技术来强制执行此安排，在 ``Child.parent_id`` 列上设置唯一约束将确保只有一个 ``Child`` 行可以引用特定的 ``Parent`` 行。

.. versionadded:: 2.0  :func:`_orm.relationship` 结构可以从给定的 :class:`_orm.Mapped` 注解导出 :paramref:`_orm.relationship.uselist` 参数的有效值。

Setting uselist=False for non-annotated configurations
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在不使用 :class:`_orm.Mapped` 注解的情况下，使用 :func:`_orm.relationship` 自动生成的一对一模式可以使用 :paramref:`_orm.relationship.uselist` 参数设置为 ``False`` 实现，如以下非注解 Declarative 示例所示：

    class Parent(Base):
        __tablename__ = "parent_table"

        id = mapped_column(Integer, primary_key=True)
        child = relationship("Child", uselist=False, back_populates="parent")


    class Child(Base):
        __tablename__ = "child_table"

        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(ForeignKey("parent_table.id"))
        parent = relationship("Parent", back_populates="child")

.. _relationships_many_to_many:

多对多
~~~~~~~~~~~~

多对多在两个类之间添加一个关联表。关联表几乎总是使用 Core :class:`_schema.Table` 对象或其他 Core 可选项，例如 :class:`_sql.Join` 对象，使用 :paramref:`_orm.relationship.secondary` 参数指示。通常，:class:`_schema.Table` 使用与 declarative 基类相关联的 :class:`_schema.MetaData` 对象，以便 :class:`_schema.ForeignKey` 指令可以找到将其与之链接的远程表::

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

.. tip::

    上述的“关联表”具有建立引用父对象和子对象的外键约束。通常情况下，每个“关联表”的 ``association.left_id`` 和 ``association.right_id`` 的数据类型都是从所引用表的数据类型推断而来的，因此可以省略。同时也建议为引用两个实体表的列进行定义：这可以通过将其定义在构成一个 **唯一约束** 或通常作为 **主键约束** 来实现；这将确保在应用程序端出现问题时，该table中不会复制的行。

设置双向多对多关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

对于双向关系，每个关系都包含一个集合。使用 :paramref:`_orm.relationship.back_populates` 指定并连接到映射对象、主键属性或同义词时，可以为每个 :func:`_orm.relationship` 指定常见的关联表：

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

后面的文章 :ref:`relationship_patterns_o2m_collection` 说明如何为多对多配置与 :ref:`relationship_patterns_o2m` 相同的集合类型。在使用 :class:`_orm.Mapped` 注解的映射中，可以通过 :class:`_orm.Mapped` 泛型类中使用的集合类型来指示集合：

    class Parent(Base):
        __tablename__ = "parent_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[Set["Child"]] = relationship(secondary=association_table)

在不使用注解的形式中，包括使用命令式映射的形式时，Python 类可通过使用 :paramref:`_orm.relationship.collection_class` 参数传递来指定要用作集合的类。

.. seealso::

    :ref:`custom_collections` - 包含有关集合配置的更多细节，包括一些将 :func:`_orm.relationship` 映射到字典的技术。

.. _relationships_many_to_many_deletion:

从多对多表中删除行
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

与 :paramref:`_orm.relationship.secondary` 参数配合使用 :func:`_orm.relationship` 时特有的一项行为是，自动将 INSERT 和 DELETE 语句应用于 :class:`.Table`，因为对象被添加或从集合中删除。**不需要手动从此表中删除**。从集合中删除记录将导致在 flush 时删除行：

    # row will be deleted from the "secondary" table
    # automatically
    myparent.children.remove(somechild)

经常出现的一个问题是，当将子对象直接交给 :meth:`.Session.delete` 时，“secondary”表中的行应如何删除。

有以下几种情况：

* 如果有一个从“Parent”到“Child”的 :func:`_orm.relationship`，但是没有相应的逆关系将特定的“child”链接到每个“Parent”，SQLAlchemy 将不会意识到当删除此特定“Child”对象时，它需要维护将其与“Parent”相关联的“secondary”表。将不会删除“secondary”表中的任何行。
* 如果存在将特定“Child”链接到每个“Parent”的关系，假设它称为 ``Child.parents``，SQLAlchemy 默认会加载 ``Child.parents`` 集合以查找所有 “Parent” 对象，并从“secondary”表中删除每行，建立此链接。请注意，此关系不需要是双向的；SQLAlchemy 严格查看与被删除的 ``Child`` 对象相关的每个 :func:`_orm.relationship`。
* 更高效的选择是使用 ON DELETE CASCADE 指令与数据库使用的外键。假设数据库支持此功能，那么当删除“child”引用行时，数据库本身可以自动删除“secondary”表中的行。在这种情况下，可以使用 :paramref:`_orm.relationship.passive_deletes` 在 :func:`_orm.relationship` 上指示 SQLAlchemy 放弃主动加载 ``Child.parents`` 集合。有关此功能的更多详细信息，请参见 :ref:`passive_deletes`。

请再次注意，这些行为仅与 ：paramref:`_orm.relationship.secondary` 和 :func:`_orm.relationship` 使用关联表有关。如果处理显式映射且当前不存在于相关 :func:`_orm.relationship` 的 :paramref:`_orm.relationship.secondary` 中的关联表，则可能会使用级联规则来自动删除根据相关实体的删除进行反应的实体 -有关此功能的更多信息请参见 :ref:`unitofwork_cascades`。

.. seealso::

    :ref:`cascade_delete_many_to_many`

    :ref:`passive_deletes_many_to_many`


.. _association_pattern:

关联对象
~~~~~~~~~~~~~~~~~~

关联对象模式是多对多的变种：当关联表包含与父表和子表（或左表和右表）的外键之外的附加列时，最理想的情况是将它们映射到自己的 ORM 映射类。此映射类映射到了会在使用多对多模式时注意到的注解参数 :paramref:`_orm.relationship.secondary` 中的 :class:`.Table`。

在关联对象模式中，不使用 :paramref:`_orm.relationship.secondary` 参数；而是将一个类直接映射到关联表。然后，通过一对多将父边与映射的关联类链接起来，再通过多对一将映射的关联类与子边链接起来，以形成一个单向的关联对象从父到关联到子的关系。对于双向关系，可以使用四个 :func:`_orm.relationship` 来将映射的关联类连接到父和两个方向的子。

下面的示例说明了一个名为 ``Association`` 的新类，其映射到名为``association`` 的 :class:`.Table`；该表现在包括一个称为“extra_data”的字符串值，该值存储与``Parent`` 和``Child`` 之间的每个关联一起使用的数据。将`Table`映射到显式类后，Parent 到Child的微不足道访问可明确使用``Association``：

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

为了说明双向关系，添加了两个更多的 :func:`_orm.relationship`，并使用 :paramref:`_orm.relationship.back_populates` 将它们链接到现有的关系上::

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

在其直接形式下使用关联对象模式需要在将子对象附加到父对象之前将其与关联实例相关联；同样，从父到子的访问可以通过关联对象进行：


    # create parent, append a child via association
    p = Parent()
    a = Association(extra_data="some data")
    a.child = Child()
    p.children.append(a)

    # iterate through child objects via association, including association
    # attributes
    for assoc in p.children:
        print(assoc.extra_data)
        print(assoc.child)

要将关联对象模式扩展到可选的直接访问 ``Association`` 对象，SQLAlchemy 提供了 :ref:`associationproxy_toplevel` 扩展。该扩展允许配置仅使用一次访问即可访问两次“跳跃”，一次从关联对象到关联对象，一次到目标属性。.. seealso::

    :ref:`associationproxy_toplevel` - 允许在父子之间直接使用“多对多”风格，
    以便进行三种类的关联对象映射。

.. warning::

  避免将关联对象模式与 :ref:`many-to-many <relationships_many_to_many>` 直接结合使用，
  因为这会在不经过特殊步骤的情况下产生可能读取和写入数据的不一致状况；
  通常使用 :ref:`association proxy <associationproxy_toplevel>` 提供更简洁的访问。
  关于通过这种组合引入的注意事项的更详细背景，请参见下一节 :ref:`association_pattern_w_m2m`。

.. _association_pattern_w_m2m:

将关联对象与多对多访问模式结合使用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如前一节所述，关联对象模式不能自动与对同一表/列进行多对多模式的使用相集成。
由此可以得出，读操作可能返回冲突数据，写操作也可能尝试写入相互冲突的更改，
从而导致完整性错误或意外的插入或删除。

为了说明，下面的示例配置了一个双向多对多关系，它将 `Parent` 和 `Child` 之间连接到 `Parent.children`
和 `Child.parents`，同时也配置了一个关联对象关系，连接到 `Parent.child_associations -> Association.child`
和 `Child.parent_associations -> Association.parent`：：

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

        # 关联 Association -> Child
        child: Mapped["Child"] = relationship(back_populates="parent_associations")

        # 关联 Association -> Parent
        parent: Mapped["Parent"] = relationship(back_populates="child_associations")


    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)

        # 与 Child 的多对多关系，绕过 `Association` 类
        children: Mapped[List["Child"]] = relationship(
            secondary="association_table", back_populates="parents"
        )

        # 关联 Parent -> Association -> Child
        child_associations: Mapped[List["Association"]] = relationship(
            back_populates="parent"
        )


    class Child(Base):
        __tablename__ = "right_table"

        id: Mapped[int] = mapped_column(primary_key=True)

        # 与 Parent 的多对多关系，绕过 `Association` 类
        parents: Mapped[List["Parent"]] = relationship(
            secondary="association_table", back_populates="children"
        )

        # 关联 Child -> Association -> Parent
        parent_associations: Mapped[List["Association"]] = relationship(
            back_populates="child"
        )

使用此 ORM 模型进行更改时，对`Parent.children`的更改不会与`Parent.child_associations`或
`Child.parent_associations`在 Python 中的更改进行协调；
虽然这些所有关系在其自身上仍将继续正常地运行，但在另一个关系进行更新之前，不会在一个关系中显示出其他关系的更改。
正常情况下，这将在:meth:`.Session.commit`之后自动触发 :class:`.Session` 的过期。

此外，如果进行相互冲突的更改，
例如添加一个新的 `Association` 对象，同时将同一关联的 `Child` 添加到 `Parent.children` 中，该过程将引发完整性错误，例如下面的示例所示：

      p1 = Parent()
      c1 = Child()
      p1.children.append(c1)

      # 多余的，将在 Association 中引发重复的 INSERT
      p1.child_associations.append(Association(child=c1))

将 `Child` 直接附加到 `Parent.children` 还意味着在不指定任何关联值的情况下在“association”表中创建行，
这会为其值接收 `NULL`。

如果你知道自己在做什么，使用像上面的映射就没什么问题；
在罕见情况下，使用 “关联对象” 模式可能会带来很多好处，
这种情况下使用多对多关系的原因可能更为重要，这是因为沿单个多对多关系加载关系更容易，
它还可以稍微优化 SQL 语句中使用“secondary”表的方式，与如何使用两个不同的关联到显式关联类相比。
但至少建议对“secondary”关系应用 :paramref:`_orm.relationship.viewonly` 参数，
以避免发生冲突的更改，同时还防止写入任何附加的关联列中的 `NULL`，如下所示：

    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)

        # 与 Child 的多对多关系，绕过 `Association` 类
        children: Mapped[List["Child"]] = relationship(
            secondary="association_table", back_populates="parents", viewonly=True
        )

        # 关联 Parent -> Association -> Child
        child_associations: Mapped[List["Association"]] = relationship(
            back_populates="parent"
        )


    class Child(Base):
        __tablename__ = "right_table"

        id: Mapped[int] = mapped_column(primary_key=True)

        # 与 Parent 的多对多关系，绕过 `Association` 类
        parents: Mapped[List["Parent"]] = relationship(
            secondary="association_table", back_populates="children", viewonly=True
        )

        # 关联 Child -> Association -> Parent
        parent_associations: Mapped[List["Association"]] = relationship(
            back_populates="child"
        )

以上映射不会将任何更改写入数据库中的 `Parent.children` 或 `Child.parents`，从而避免冲突的写入。
但是，在读取 `Parent.children` 或 `Child.parents`时，并不一定与从 `Parent.child_associations` 或
`Child.parent_associations`中读取的数据匹配，如果正在与在同一个事务或:class:`.Session`中更改这些集合，则这些集合将不匹配。
如果关联对象关系的使用不太频繁，并且仔细组织了针对访问多对多集合的代码以避免过时读取
(在极端情况下，在当前事务中直接使用 :meth:`_orm.Session.expire` 直接刷新集合)，
则该模式可能是可行的。

替代以上模式的一个流行选择是，将直接的多对多关系 `Parent.children` 和 `Child.parents` 替换为将透明地代理通过 `Association` 类，
同时从 ORM 的角度保持一切一致的扩展。该扩展称为 :ref:`Association Proxy <associationproxy_toplevel>`。

.. seealso::

    :ref:`associationproxy_toplevel` - 允许在父子之间直接使用“多对多”风格，
    以便进行三种类的关联对象映射。

.. _orm_declarative_relationship_eval:

关系参数的延迟评估
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

上一节中大多数示例说明了如何使用其目标类的字符串名称来引用各种 :func:`_orm.relationship` 构造，而不是类本身，
例如，使用 :class:`_orm.Mapped` 时，会生成仅作为字符串存在的正向引用比如说：

    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(back_populates="parent")


    class Child(Base):
        # ...

        parent: Mapped["Parent"] = relationship(back_populates="children")

同样，使用未注释的表单，例如未注释的 Declarative 或 Imperative 映射， :func:`_orm.relationship` 构造也支持直接字符串名称的传递::

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

这些字符串名称会在映射解析阶段中被解析为类，该阶段通常在定义所有映射之后触发，并且通常由映射本身的第一个使用触发。 
:class:`_orm.registry` 对象是存储这些名称并将其解析为所指向的映射类的容器。

除了 :func:`_orm.relationship` 的主要类参数之外，还可以指定取决于未定义类上存在的列的其他参数，
这些列通常也可以指定为 Python 函数或更常见的是字符串。对于除主参数外的大多数参数，字符串输入将被 **作为 Python 表达式使用 Python 内置的 eval() 函数来计算**，因为它们旨在接收完整的 SQL 表达式。

.. warning::大家要知道由于Python interprets the late-evaluated string arguments passed to the :func:`_orm.relationship` mapper, 
参数也应被设计为不接收不可信用户输入。 `eval()` 对不受信任的用户输入不是**安全的**。

在此评估中使用的全名词空间包括为此声明基础中的所有映射类，以及 `sqlalchemy` 包的内容，包括表达式函数，例如 `func()` 等：

    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(
            order_by="desc(Child.email_address)",
            primaryjoin="Parent.id == Child.parent_id",
        )

如果同一名称的类存在于多个模块中，则可以在任何这些字符串表达式中将字符串类名指定为模块限定路径。

例如，为了区分 `myapp.model1.Child` 和 `myapp.model2.Child`，我们可以指定 `model1.Child` 或 `model2.Child`：

    class Parent(Base):
        # ...

        children: Mapped[List["Child"]] = relationship(
            "model1.Child",
            order_by="desc(model1.Child.email_address)",
            primaryjoin="Parent.id == model1.Child.parent_id",
        )

在上面的示例中，可以直接传递类位置字符串到 :paramref:`_orm.relationship.argument` 中，从而将给定的 :class:`.Table` 对象通过名称解析为 Python 表达式。

下面是一个仅导入类型的例子，其中将结合运行时指定的目标类搜索 :class:`_orm.registry` 来搜索包含在此内的正确名称：

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

任何在这些函数/ lambda 或字符串之一中作为参数传递的参数，如果有多个同名的 :class:`_schema.Table` 对象，则会被解释为它们所指向的 :class:`_schema.MetaData` 集合中的标识符名称。

.. warning::

    如上所述，传递给 :func:`_orm.relationship` 的上述参数是使用 eval()评估为 Python 代码表达式的。
    **不要向这些参数传递不受信任的输入。**

在声明完成后向映射类添加关系
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

要注意的是，只要声明完整的映射模型，就可以在任何时候添加映射到声明性基础中的 :class:`_orm.MapperProperty`，
并且注释表单中注释的形式不受支持。如果想在 `Address` 类可用后，实现此 :func:`_orm.relationship`，我们还可以在之后应用它：

# 首先，在 Child 还没有被创建的模块 A 中，
# 我们创建一个 Parent 类，它对 Child 一无所知

    class Parent(Base):
        ...


    # ... 然后，将模块 B 导入模块 A 之后：

    class Child(Base):
        ...

    from module_a import Parent

    # 将 User.addresses 关系作为类变量分配。 
    # 声明性基类将拦截此操作并映射关系。
    Parent.children = relationship(Child, primaryjoin=Child.parent_id == Parent.id)

与 ORM 映射的列一样，在声明完成后，可以在任何时候向映射类添加映射属性；
因此，相关的类必须在 :func:`_orm.relationship` 构造中直接指定为类本身、类的字符串名称或可返回到目标类的引用的可调用函数。

.. note::

    如前所述，在已映射的类中分配映射属性仅在使用“声明性基础类”时才能正确地运作，
    这意味着用户定义的 :class:`_orm.DeclarativeBase` 子类或通过 :func:`_orm.declarative_base` 
    或者 :meth:`_orm.registry.generate_base` 返回的动态生成的 class。
    此“基”类包括实现了特殊 ``__setattr__()`` 方法的 Python 元类，该方法截取了这些操作。

    如果使用诸如 :meth:`_orm.registry.mapped` 或像 :meth:`_orm.registry.map_imperatively` 的命令式函数之类的函数映射来映射类，
    则运行时分配映射属性到已映射的类就**不会**正常工作。

.. _orm_declarative_relationship_secondary_eval:

在多对多中使用延迟评估的 "secondary" 参数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

多对多关系使用 :paramref:`_orm.relationship.secondary` 参数，该参数通常指示引用通常未映射的 :class:`_schema.Table` 对象或其他 Core 可选择对象。
此处支持使用可调用的 lambda 或字符串名称的延迟评估，其中字符串解析的 Python 表达式会将标识符名称链接到同名 :class:`_schema.Table` 对象，
这些对象存在于由当前 :class:`_orm.registry` 指向的相同的 :class:`_schema.MetaData` 集合中。

对于在 :ref:`relationships_many_to_many` 中给出的示例，如果我们假设“association_table” :class:`.Table` 对象在比映射类本身晚定义的时间点上将会定义，
我们可以使用 lambda 表示 :func:`_orm.relationship`：

    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(
            "Child", secondary=lambda: association_table
        )

或使用名称定位同一 :class:`.Table` 对象，名称为 :class:`.Table`：

    class Parent(Base):
        __tablename__ = "left_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        children: Mapped[List["Child"]] = relationship(secondary="association_table")


.. warning::

    当作为字符串传递时， :paramref:`_orm.relationship.secondary` 参数是使用 Python 的 `eval()` 函数计算的，
    即时通常是表的名称。 **不要将不可信用户输入传递给此字符串**。