.. _self_referential:

邻接列表关系
---------------------------

**邻接列表**模式是一种常见的关系模式，其中一个表包含对自身的外键引用，换句话说是一种**自引用关系**。这是用平坦的表表示分层数据的最常见方法。其他方法包括 **嵌套集**，有时称为“修改先序遍历”，以及 **物化路径**。尽管修改先序遍历在评估其在SQL查询中的流畅性时具有吸引力，但邻接列表模型可能是大多数分层存储需求的最合适模式，原因是并发性、减少复杂性以及修改先序遍历在可以完全加载子树到应用程序空间的应用程序上几乎没有优势。

.. seealso::

    本节详细介绍了自引用关系的单表版本。有关使用第二个表作为关联表的自引用关系，请参见   :ref:`self_referential_many_to_many`  节。

在本示例中，我们将使用一个名为“Node”的单个映射类，表示树形结构::

    class Node(Base):
        __tablename__ = "node"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("node.id"))
        data = mapped_column(String(50))
        children = relationship("Node")

有了这个结构，如下所示的图表:

.. sourcecode:: text

    root --+---> child1
           +---> child2 --+--> subchild1
           |              +--> subchild2
           +---> child3

将表示为以下数据:

.. sourcecode:: text

    id       parent_id     data
    ---      -------       ----
    1        NULL          root
    2        1             child1
    3        1             child2
    4        3             subchild1
    5        3             subchild2
    6        1             child3

此处的   :func:`_orm.relationship`  配置与“正常”的一对多关系相同，唯一的区别在于“方向”，即关系是一对多还是多对一，默认情况下假定为一对多。要将关系建立为多对一，需要添加额外的指令，称为  :paramref:` _orm.relationship.remote_side` ，它是一个   :class:`_schema.Column`  或一个   :class:` _schema.Column`  对象集合，表示那些应被视为“远程”的对象：

    class Node(Base):
        __tablename__ = "node"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("node.id"))
        data = mapped_column(String(50))
        parent = relationship("Node", remote_side=[id])

在上述例子中，将``id``列应用为``parent``函数的  :paramref:`_orm.relationship.remote_side` ，因此将` `parent_id``视为“本地”侧面，关系然后表现为多对一。

像往常一样，可以通过两个   :func:`_orm.relationship`  结构相互连接，这两个结构之间可以使用  :paramref:` _orm.relationship.back_populates`  形成双向关系：

    class Node(Base):
        __tablename__ = "node"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("node.id"))
        data = mapped_column(String(50))
        children = relationship("Node", back_populates="parent")
        parent = relationship("Node", back_populates="children", remote_side=[id])

.. seealso::

      :ref:`examples_adjacencylist`  - 此工作示例已更新为SQLAlchemy 2.0

复合邻接列表
~~~~~~~~~~~~~~~~~~~~~~~~~

邻接列表关系的子类别是一种特殊情况，即在联接条件的“本地”和“远程”侧上都存在特定列。下面是一个 ``Folder`` 类的示例；使用复合主键， ``account_id``
列引用自身，以指示与父级相同帐户中的子文件夹；而``folder_id``引用该帐户中的特定文件夹：

    class Folder(Base):
        __tablename__ = "folder"
        __table_args__ = (
            ForeignKeyConstraint(
                ["account_id", "parent_id"], ["folder.account_id", "folder.folder_id"]
            ),
        )

        account_id = mapped_column(Integer, primary_key=True)
        folder_id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer)
        name = mapped_column(String)

        parent_folder = relationship(
            "Folder", back_populates="child_folders", remote_side=[account_id, folder_id]
        )

        child_folders = relationship("Folder", back_populates="parent_folder")

在上述示例中，将 ``account_id`` 传递到  :paramref:`_orm.relationship.remote_side`  列表中。 ` `relationship`` 函数识别到在此处 ``account_id`` 列在两侧，并将“远程”列与它识别为唯一存在于“远程”侧的 ``folder_id`` 列。

.. _self_referential_query:

自引用查询策略
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

自引用结构的查询与任何其他查询一样：

    # 获取名称为“child2”的所有节点
    session.scalars(select(Node).where(Node.data == "child2"))

但是，在尝试沿着树的一级到下一级的外键连接时，需要特别小心。在SQL中，从表连接到自身需要将表达式的至少一侧“别名”，以便可以确切地引用它。

回顾一下 ORM 教程中的   :ref:`orm_queryguide_orm_aliases` ，通常使用   :func:` _orm.aliased`  结构为 ORM 实体提供“别名”。使用这种技术将 ``Node`` 连接到自身如下所示：

.. sourcecode:: python+sql

    from sqlalchemy.orm import aliased

    nodealias = aliased(Node)
    session.scalars(
        select(Node)
        .where(Node.data == "subchild1")
        .join(Node.parent.of_type(nodealias))
        .where(nodealias.data == "child2")
    ).all()
    {execsql}SELECT node.id AS node_id,
            node.parent_id AS node_parent_id,
            node.data AS node_data
    FROM node JOIN node AS node_1
        ON node.parent_id = node_1.id
    WHERE node.data = ?
        AND node_1.data = ?
    ['subchild1', 'child2']


.. _self_referential_eager_loading:

配置自引用关系的急加载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

关系的快速加载是通过从父到子表使用连接或外连接进行的，以便可以从单个SQL语句或所有立即子代集合的第二个语句填充父及其即时子集合或引用。SQLAlchemy 在连接到相关项目时在所有情况下使用别名表，因此与自引用连接兼容。但是，要使用自引用关系的急加载，需要告诉SQLAlchemy它应该加入和/或查询多少级;否则，快速加载将不会发生。通过  :paramref:`~.relationships.join_depth`  配置此深度设置：

.. sourcecode:: python+sql

    class Node(Base):
        __tablename__ = "node"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("node.id"))
        data = mapped_column(String(50))
        children = relationship("Node", lazy="joined", join_depth=2)


    session.scalars(select(Node)).all()
    {execsql}SELECT node_1.id AS node_1_id,
            node_1.parent_id AS node_1_parent_id,
            node_1.data AS node_1_data,
            node_2.id AS node_2_id,
            node_2.parent_id AS node_2_parent_id,
            node_2.data AS node_2_data,
            node.id AS node_id,
            node.parent_id AS node_parent_id,
            node.data AS node_data
    FROM node
        LEFT OUTER JOIN node AS node_2
            ON node.id = node_2.parent_id
        LEFT OUTER JOIN node AS node_1
            ON node_2.id = node_1.parent_id
    []