.. currentmodule:: sqlalchemy.orm

.. _mapper_composite:

复合列类型
======================

一组列可以与一个用户定义的数据类型相关联，现代用法通常是一个 Python dataclass_。ORM
提供了一个单一的属性来表示使用您提供的类的一组列。

一个简单的例子表示   :class:`_types.Integer`  列对的 ` `Point`` 对象，使用属性 ``.x`` 和 ``.y``。
使用一个 dataclass，这些属性使用相应的 ``int`` Python 类型进行定义::

    import dataclasses


    @dataclasses.dataclass
    class Point:
        x: int
        y: int

也支持非 dataclass 形式，但需要实现额外的方法。关于使用不是 dataclass 类的示例，请参见本节的部分
  :ref:`composite_legacy_no_dataclass` 。

.. versionadded:: 2.0 The   :func:`_orm.composite`  construct fully supports
   Python dataclasses including the ability to derive mapped column datatypes
   from the composite class.

我们将创建一个映射到表 ``vertices`` 的映射，它表示两个点
可以作为 ``x1/y1`` 和 ``x2/y2``。 ``Point`` 类与之关联
使用   :func:`_orm.composite`  构造映射列。

下面示例说明了最现代的形式   :func:`_orm.composite` ，如
与完全使用的   :ref:`Annotated Declarative Table <orm_declarative_mapped_column>`  配置。
每个构成列的   :func:`_orm.mapped_column`  构造传递到   :func:` _orm.composite` ，
指示要生成的列的零个或多个方面，在本例中为名称；

  :func:`_orm.composite`  构造从 dataclass 直接派生列类型（在本例中为 ` `int``，对应于   :class:`_types.Integer` ）::

    from sqlalchemy.orm import DeclarativeBase, Mapped
    from sqlalchemy.orm import composite, mapped_column


    class Base(DeclarativeBase):
        pass


    class Vertex(Base):
        __tablename__ = "vertices"

        id: Mapped[int] = mapped_column(primary_key=True)

        start: Mapped[Point] = composite(mapped_column("x1"), mapped_column("y1"))
        end: Mapped[Point] = composite(mapped_column("x2"), mapped_column("y2"))

        def __repr__(self):
            return f"Vertex(start={self.start}, end={self.end})"

上述映射将对应一个 CREATE TABLE 语句：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy.schema import CreateTable
    >>> print(CreateTable(Vertex.__table__))
    {printsql}CREATE TABLE vertices (
      id INTEGER NOT NULL,
      x1 INTEGER NOT NULL,
      y1 INTEGER NOT NULL,
      x2 INTEGER NOT NULL,
      y2 INTEGER NOT NULL,
      PRIMARY KEY (id)
    )


使用映射的复合列类型
------------------------------------------------------

使用前面部分中说明的映射，我们可以处理
“Vertex” 类，其中 ``.start`` 和 ``.end`` 属性
自动透明地引用 ``Point`` 类所指向的列，以及涉及 ``Vertex`` 类实例的情况，其中 ``.start`` 和
``.end`` 属性将引用 ``Point`` 类实例。``x1``,
``y1``, ``x2``, 和 ``y2`` 列被透明地处理：

* **持久化 Point 对象**

  我们可以创建一个 ``Vertex`` 对象，将 ``Point`` 对象分配为成员，
  并按预期进行持久化：

  .. sourcecode:: pycon+sql

    >>> v = Vertex(start=Point(3, 4), end=Point(5, 6))
    >>> session.add(v)
    >>> session.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO vertices (x1, y1, x2, y2) VALUES (?, ?, ?, ?)
    [generated in ...] (3, 4, 5, 6)
    COMMIT

* **选择其他 Point 对象作为列**

   :func:`_orm.composite`  将允许 ` `Vertex.start`` 和 ``Vertex.end``
  行为像单个 SQL 表达式一样，在使用 ORM 的情况下尽可能多地使用   :class:`_orm.Session`  （包括旧版
    :class:`_orm.Query`  对象）来选择 ` `Point`` 对象：

  .. sourcecode:: pycon+sql

    >>> stmt = select(Vertex.start, Vertex.end)
    >>> session.execute(stmt).all()
    {execsql}SELECT vertices.x1, vertices.y1, vertices.x2, vertices.y2
    FROM vertices
    [...] ()
    {stop}[(Point(x=3, y=4), Point(x=5, y=6))]

* **在 SQL 表达式中比较 Point 对象**

  ``Vertex.start`` 和 ``Vertex.end`` 属性可用于
  根据需要使用临时的 "Point" 对象进行 WHERE 条件等，用于比较：

  .. sourcecode:: pycon+sql

    >>> stmt = select(Vertex).where(Vertex.start == Point(3, 4)).where(Vertex.end < Point(7, 8))
    >>> session.scalars(stmt).all()
    {execsql}SELECT vertices.id, vertices.x1, vertices.y1, vertices.x2, vertices.y2
    FROM vertices
    WHERE vertices.x1 = ? AND vertices.y1 = ? AND vertices.x2 < ? AND vertices.y2 < ?
    [...] (3, 4, 7, 8)
    {stop}[Vertex(Point(x=3, y=4), Point(x=5, y=6))]

  .. versionadded:: 2.0    :func:`_orm.composite`  constructs now support
     "ordering" comparisons such as ``<``, ``>=``, and similar, in addition
     to the already-present support for ``==``, ``!=``.

  .. tip::  上面使用“less than”运算符（``<``）进行的“次序”比较以及使用 ``==`` 进行的“等于”比较，
     在生成 SQL 表达式时，是由   :class:`_orm.Composite.Comparator`  类实现的，而不是使用复合类
     本身的比较方法，例如``__lt__()``或``__eq__()``方法。从这里可以得出
     得出，上面的``Point`` dataclass 不需要为上面的 SQL 操作实现 dataclasses ``order = True``
     参数。   :ref:`composite_operations`  部分包含如何自定义比较操作的背景信息。

* **更新 Vertex 实例中的 Point 对象**

  默认情况下，必须将 ``Point`` 对象 **替换为新对象** 才能检测到更改：

  .. sourcecode:: pycon+sql

    >>> v1 = session.scalars(select(Vertex)).one()
    {execsql}SELECT vertices.id, vertices.x1, vertices.y1, vertices.x2, vertices.y2
    FROM vertices
    [...] ()
    {stop}

    >>> v1.end = Point(x=10, y=14)
    >>> session.commit()
    {execsql}UPDATE vertices SET x2=?, y2=? WHERE vertices.id = ?
    [...] (10, 14, 1)
    COMMIT

  为便于在复合对象上进行现场更改，必须使用   :ref:`mutable_toplevel`  扩展。请参见部分
    :ref:`mutable_composites`  的示例。

.. _orm_composite_other_forms:

其他复合形式的映射
----------------------------------

  :func:`_orm.composite`  构造可以使用   :func:` _orm.mapped_column`  构造，一个   :class:`_schema.Column` ，或者现有映射的字符串名称等相关列。
下面的示例说明了与主要部分相同的等效映射形式。

* 直接映射列，然后将其传递给复合

  在这里，我们将现有的   :func:`_orm.mapped_column`  实例传递给
    :func:`_orm.composite`  构造，如下例所示，其中还会传递 ` `Point`` 类，作为   :func:`_orm.composite` 
  的第一个参数::

    from sqlalchemy import Integer
    from sqlalchemy.orm import mapped_column, composite


    class Vertex(Base):
        __tablename__ = "vertices"

        id = mapped_column(Integer, primary_key=True)
        x1 = mapped_column(Integer)
        y1 = mapped_column(Integer)
        x2 = mapped_column(Integer)
        y2 = mapped_column(Integer)

        start = composite(Point, x1, y1)
        end = composite(Point, x2, y2)

* 直接映射列，将属性名称传递给 composite

  我们可以使用更详细的注释形式编写上述示例，其中我们可以选择属性名称传递到   :func:`_orm.composite`  而不是
  完整的列结构::

    from sqlalchemy.orm import mapped_column, composite, Mapped


    class Vertex(Base):
        __tablename__ = "vertices"

        id: Mapped[int] = mapped_column(primary_key=True)
        x1: Mapped[int]
        y1: Mapped[int]
        x2: Mapped[int]
        y2: Mapped[int]

        start: Mapped[Point] = composite("x1", "y1")
        end: Mapped[Point] = composite("x2", "y2")

* 命令式映射和命令式表

  当使用   :ref:`imperative table <orm_imperative_table_configuration>`  或完全   :ref:` imperative <orm_imperative_mapping>`  映射时，我们可以直接访问   :class:`_schema.Column`  对象。
  这些也可以传递给   :func:`_orm.composite` ，如下面的命令示例所示::

     mapper_registry.map_imperatively(
         Vertex,
         vertices_table,
         properties={
             "start": composite(Point, vertices_table.c.x1, vertices_table.c.y1),
             "end": composite(Point, vertices_table.c.x2, vertices_table.c.y2),
         },
     )

.. _composite_legacy_no_dataclass:

使用旧版非数据类
----------------------------


如果不使用 dataclass，则自定义数据类型类的要求为
它的构造函数需要接受与其基于列的格式相对应的位置参数，
    并提供一个方法 ``__composite_values__()``，返回对象状态的列表或元组，按其基于列的属性顺序。它
还应提供充分的 ``__eq__()`` 和 ``__ne__()`` 方法来测试两个实例的等式。

为说明主要部分中相当的“Point”类，而不使用 dataclass：

    class Point:
        def __init__(self, x, y):
            self.x = x
            self.y = y

        def __composite_values__(self):
            return self.x, self.y

        def __repr__(self):
            return f"Point(x={self.x!r}, y={self.y!r})"

        def __eq__(self, other):
            return isinstance(other, Point) and other.x == self.x and other.y == self.y

        def __ne__(self, other):
            return not self.__eq__(other)

使用   :func:`_orm.composite`  进行下一步处理，然后与列相关联
使用: ref:`orm_composite_other_forms` 中的一个表单。

追踪复合对象上的现场变化
-----------------------------------------

现场更改现有复合值不会自动跟踪。而是需要复合类明确向其父对象提供事件。这个任务在很大程度上被自动化了
通过使用   :class:`.MutableComposite`  mixin，它使用事件
将每个用户定义的复合对象与所有父关联关联起来。请参见部分
  :ref:`mutable_composites`  中的示例。

.. _composite_operations:

为复合重新定义比较操作
-----------------------------------------------

“等于”比较操作默认情况下会产生对所有相应列进行等式测试的 AND。这可以使用
``comparator_factory`` 参数   :func:`.composite`  来更改，在其中我们
指定自定义   :class:`.CompositeProperty.Comparator`  类
来定义现有或新的操作。
下面我们演示“greater than”运算符，实现
与基本“greater than”相同的表达式::

    import dataclasses

    from sqlalchemy.orm import composite
    from sqlalchemy.orm import CompositeProperty
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.sql import and_


    @dataclasses.dataclass
    class Point:
        x: int
        y: int


    class PointComparator(CompositeProperty.Comparator):
        def __gt__(self, other):
            """redefine the 'greater than' operation"""

            return and_(
                *[
                    a > b
                    for a, b in zip(
                        self.__clause_element__().clauses,
                        dataclasses.astuple(other),
                    )
                ]
            )


    class Base(DeclarativeBase):
        pass


    class Vertex(Base):
        __tablename__ = "vertices"

        id: Mapped[int] = mapped_column(primary_key=True)

        start: Mapped[Point] = composite(
            mapped_column("x1"), mapped_column("y1"), comparator_factory=PointComparator
        )
        end: Mapped[Point] = composite(
            mapped_column("x2"), mapped_column("y2"), comparator_factory=PointComparator
        )

由于 ``Point`` 是 dataclass，我们可以使用
``dataclasses.astuple()`` 来获取 ``Point`` 实例的元组形式。

然后自定义比较器返回适当的 SQL 表达式：

.. sourcecode:: pycon+sql

  >>> print(Vertex.start > Point(5, 6))
  {printsql}vertices.x1 > :x1_1 AND vertices.y1 > :y1_1


嵌套复合对象
-------------------

可以通过重定义复合类内的行为来简单嵌套复合对象，在使用   :func:`_orm.composite`  时将其映射到完整列的长度通常会更好。
在这种情况下，需要定义另外的方法来在“嵌套”和“扁平”形式之间移动。

接下来，我们将重新组织 ``Vertex`` 类，使其成为参照 ``Point`` 对象的复合对象。 ``Vertex`` 和 ``Point`` 可以是 dataclasses，
但是我们将添加一个自定义构建方法来 ``Vertex``，以便可以通过传递值到 ``Vertex._generate()`` 方法来创建新的 ``Vertex`` 对象。

我们还将实现 ``__composite_values__()`` 方法，这是   :func:`_orm.composite`  构造（在之前的部分介绍的）可以识别的一个固定名称，指示标准接收对象作为列值的方法一元组，在这种情况下通常会替代基于 dataclass 的方法论。

具有自定义 ``_generate()`` 构造函数和
``__composite_values__()`` serializer 方法，我们现在可以在水平列和``Vertex`` 包含 ``Point`` 实例的对象之间移动。
``Vertex._generate`` 方法作为   :func:`_orm.composite`  的第一个参数传递，作为新的源
``Vertex`` 实例，并且``__composite_values__()`` 方法将隐式地使用   :func:`_orm.composite` 。

对于示例，复合``Vertex`` 然后映射到名为``HasVertex`` 的类中，
其中包含四个源列的   :class:`.Table`  所在之处：

    from __future__ import annotations

    import dataclasses
    from typing import Any
    from typing import Tuple

    from sqlalchemy.orm import composite
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    @dataclasses.dataclass
    class Point:
        x: int
        y: int


    @dataclasses.dataclass
    class Vertex:
        start: Point
        end: Point

        @classmethod
        def _generate(cls, x1: int, y1: int, x2: int, y2: int) -> Vertex:
            """generate a Vertex from a row"""
            return Vertex(Point(x1, y1), Point(x2, y2))

        def __composite_values__(self) -> Tuple[Any, ...]:
            """generate a row from a Vertex"""
            return dataclasses.astuple(self.start) + dataclasses.astuple(self.end)


    class Base(DeclarativeBase):
        pass


    class HasVertex(Base):
        __tablename__ = "has_vertex"
        id: Mapped[int] = mapped_column(primary_key=True)
        x1: Mapped[int]
        y1: Mapped[int]
        x2: Mapped[int]
        y2: Mapped[int]

        vertex: Mapped[Vertex] = composite(Vertex._generate, "x1", "y1", "x2", "y2")

上述映射可根据 ``HasVertex``、``Vertex`` 和使用::

    hv = HasVertex(vertex=Vertex(Point(1, 2), Point(3, 4)))

    session.add(hv)
    session.commit()

    stmt = select(HasVertex).where(HasVertex.vertex == Vertex(Point(1, 2), Point(3, 4)))

    hv = session.scalars(stmt).first()
    print(hv.vertex.start)
    print(hv.vertex.end)

.. _dataclass: https://docs.python.org/3/library/dataclasses.html

复合API
-------------

.. autofunction:: composite