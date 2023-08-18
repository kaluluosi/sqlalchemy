.. _self_referential:

邻接列表关系
------------

**邻接列表模式**是一种常见的关系模式，其中一个表包含对本身的外键引用，也就是**自引用关系**。这是在平面表中表示分层数据的最常见方式。
其他方法包括**嵌套集**(有时称为“修改遍历”)，以及**材料化路径**。尽管从在 SQL 查询中的流畅度来衡量修改遍历的吸引力，但邻接列表模型很可能是大多数分层存储需求中最合适的模式，因为它具有并发性、简化性和修改遍历对于能够完全将子树加载到应用程序空间的应用来说没有优势的原因。

.. 参见::

    这节详细介绍了自引用关系的单表版。要查看使用第二个表作为关联表的自引用关系，请参见 :ref:`self_referential_many_to_many`。

在这个例子中，我们将使用一个单一映射的名为 ``Node`` 的类来表示树结构:

.. sourcecode:: python

    class Node(Base):
        __tablename__ = "node"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("node.id"))
        data = mapped_column(String(50))
        children = relationship("Node")

对于上述结构，类似以下形式的图：

.. sourcecode:: text

    root --+---> child1
           +---> child2 --+--> subchild1
           |              +--> subchild2
           +---> child3

可以用以下数据来表示：

.. sourcecode:: text

    id       parent_id     data
    ---      -------       ----
    1        NULL          root
    2        1             child1
    3        1             child2
    4        3             subchild1
    5        3             subchild2
    6        1             child3

:func:`_orm.relationship` 在这里的配置方式与 "通常" 的一对多关系相同，唯一的
例外是默认情况下假定关系是一对多还是多对一 。为了将关系建立为多对一，需要添加一个额外指令 :paramref:`_orm.relationship.remote_side`。它是一个 :class:`_schema.Column` 或 :class:`_schema.Column` 对象的集合，表示应该视为 "远程" 的内容。

.. sourcecode:: python

    class Node(Base):
        __tablename__ = "node"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("node.id"))
        data = mapped_column(String(50))
        parent = relationship("Node", remote_side=[id])


上述代码将 ``id`` 指定为 ``parent`` 的 ``remote_side``，从而将 ``parent_id`` 视为“本地”方向，然后关系的行为就会变为多对一。

同样，两个方向可以结合为双向关系，使用由 :paramref:`_orm.relationship.back_populates` 关联的两个 :func:`_orm.relationship` 构造来链接:

.. sourcecode:: python

    class Node(Base):
        __tablename__ = "node"
        id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(Integer, ForeignKey("node.id"))
        data = mapped_column(String(50))
        children = relationship("Node", back_populates="parent")
        parent = relationship("Node", back_populates="children", remote_side=[id])

.. 参见::

    : ref：`examples_adjacencylist`-适用于 SQLAlchemy 2.0 的工作示例

复合邻接列表
~~~~~~~~~~~~~~~~~~~~~~~~~

邻接列表关系的一个子类别是某些情况下为本地和远程连接条件都存在的表添加特定的列的情况。
下面是 ``Folder`` 类的一个示例，使用复合主键，``account_id`` 列引用它自身，以表示与父级在同一帐户中的子文件夹, ``folder_id``则表示该帐户中特定的文件夹:

.. sourcecode:: python

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

上面，将 ``account_id`` 传递到 :paramref:`_orm.relationship.remote_side` 列表中。 :func:`_orm.relationship` 识别到，在两侧都存在 ``account_id`` 时
，将“远程”列与 ``folder_id`` 列对齐，后者被认为是仅出现在“远程”方向上的列。

.. _self_referential_query:

自引用查询策略
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

查询自引用结构的工作方式与任何其他查询相同:

.. sourcecode:: python

    # 获取名为 'child2' 的所有节点
    session.scalars(select(Node).where(Node.data == "child2"))


但是，在尝试从一级树到上一级树模式上连接的时候需要格外小心。在SQL中，从表连接自身至少会要求将表达式的某一侧"别名"，以便可以无歧义地引用它。

还记得教程中在 ORM 中使用 :func:`_orm.aliased` 的代码吗？我们通常使用它来提供ORM实体的“别名”。使用此技术从 ``Node`` 连接到自身将如下所示：

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

配置自引用关系的预加载
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用连接或外连接从父表到子表进行关系的即时加载，使得可以从单个 SQL 语句或者所有即时子集的第二个语句来填充父级和其子级引用 or 集合。SQLAlchemy 的并入和子查询即时加载在连接到关联项时，在所有情况下都使用别名表，因此与自引用关联性的连接可以兼容。但是，为了使用自引用关系的即时加载，SQLAlchemy 需要告诉它需要连接和/或查询多深的层级；否则，即时加载将根本不会发生。该深度设置通过 :paramref:`~.relationships.join_depth` 进行配置。

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