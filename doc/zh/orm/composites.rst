.. currentmodule:: sqlalchemy.orm

.. _mapper_composite:

组合列类型
======================

一组列可以关联到单个用户定义的datatype，现代使用中通常是一个Python dataclass_。ORM提供了一个属性，该属性使用您提供的类表示列组。

一个简单的例子将:class:`_types.Integer`列对表示为一个``Point``对象，该对象具有``.x``和``.y``属性。使用dataclass，这些属性使用相应的``int``Python类型定义::

    import dataclasses


    @dataclasses.dataclass
    class Point:
        x: int
        y: int

非dataclass形式也被接受，但需要实现额外的方法。有关使用非数据类类的示例，请参见本节的部分:ref:`composite_legacy_no_dataclass` 。

.. versionadded:: 2.0 The :func:`_orm.composite` construct fully supports
   Python dataclasses including the ability to derive mapped column datatypes
   from the composite class.

我们将创建一个映射到表``vertices``的映射，该表将两个点表示为``x1 / y1``和``x2 / y2``。使用:func:`_orm.composite`构造将``Point``类与映射的列相关联。

下面的示例说明了最新形式的:func:`_orm.composite`，如何与完全使用有注释的声明性表格Annotated Declarative Table<orm_declarative_mapped_column>`进行配置。直接传递代表每个列的:func:`_orm.mapped_column`构造函数到:func:`_orm.composite`中，表示要生成的列的零个或多个方面，在本例中，:func:`_orm.composite`构造函数将从数据类直接导出列类型（在这种情况下是``int``，对应``_types.Integer``）::

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

上面的映射将对应于CREATE TABLE语句：

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


使用已映射的组合列类型
-------------------------------------------

使用如上节所示的映射，我们可以使用``Vertex``类，其中``.start``和``.end``属性将透明地引用由``Point``类引用的列，以及``Vertex``类的实例，其中``.start``和``. end``属性将引用``Point``类的实例。``x1`，``y1``，``x2``和``y2``列会自动处理：

* **持续的Point对象**

  我们可以创建一个``Vertex``对象，将``Point``对象分配为成员，并且它们将正常保存：

  .. sourcecode:: pycon+sql

    >>> v = Vertex(start=Point(3, 4), end=Point(5, 6))
    >>> session.add(v)
    >>> session.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO vertices (x1, y1, x2, y2) VALUES (?, ?, ?, ?)
    [generated in ...] (3, 4, 5, 6)
    COMMIT

* **作为列选择Point对象**

  :func:`_orm.composite`允许``Vertex.start``和``Vertex.end``属性尽可能地在使用ORM:class:`_orm.Session`（包括遗留的:class:`_orm.Query`对象）选择``Point``对象时，
  像一个单一的SQL表达式：

  .. sourcecode:: pycon+sql

   >>> stmt = select(Vertex.start, Vertex.end)
   >>> session.execute(stmt).all()
   {execsql}SELECT vertices.x1, vertices.y1, vertices.x2, vertices.y2
   FROM vertices
   [...] ()
   {stop}[(Point(x=3, y=4), Point(x=5, y=6))]

* **在SQL表达式中比较Point对象**

  可以使用“Point”对象进行WHERE条件和类似操作的"Vertex.start" 和“Vertex.end”属性，将特定的“Point”对象用于比较：

  .. sourcecode:: pycon+sql

    >>> stmt = select(Vertex).where(Vertex.start == Point(3, 4)).where(Vertex.end < Point(7, 8))
    >>> session.scalars(stmt).all()
    {execsql}SELECT vertices.id, vertices.x1, vertices.y1, vertices.x2, vertices.y2
    FROM vertices
    WHERE vertices.x1 = ? AND vertices.y1 = ? AND vertices.x2 < ? AND vertices.y2 < ?
    [...] (3, 4, 7, 8)
    {stop}[Vertex(Point(x=3, y=4), Point(x=5, y=6))]

  .. versionadded:: 2.0  :func:`_orm.composite` constructs now support
     "ordering" comparisons such as ``<``, ``>=``, and similar, in addition
     to the already-present support for ``==``, ``!=``.

  .. tip::  上述使用“小于”运算符（``<``）以及使用“相等性”比较的“Vertex.start”和“Vertex.end”操作，当用于生成时SQL表达式，由:class:`_orm.Composite.Comparator`
     类实现，并不使用复合类本身的比较方法，例如“__lt __（）”或“__eq __（）”方法。因此，可以将上面的“Point”数据类实现为不需要进行SQL运算 的“order = True”参数。有关如何自定义比较操作的背景信息，请参见:ref:`composite_operations`。

* **在Vertex实例上更新Point对象**

  默认情况下，必须使用新对象**替换Point对象* **** ****才能检测到更改：

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

  为了允许在复合对象上进行原地更改，必须使用:ref:`mutable_toplevel`扩展。有关示例，请参见:ref:`mutable_composites` 中的部分。

.. _orm_composite_other_forms:

其他用于复合的映射形式
----------------------------------

：func:`_orm.composite`构造函数可以使用：func:`~sqlalchemy.orm.mapped_column`构造函数，:class:`~sqlalchemy.schema.Column`或映射列的字符串名称来传递相关的列。下面的示例说明了等效的映射，就像第一个代码单元（"Composite Column Types"）一样。

* 直接映射列，然后传递给composite

  这里我们将现有的:func:`~sqlalchemy.orm.mapped_column`实例传递到:func:`_orm.composite`构建函数中，
  就像在下面的无法注释的示例中那样，我们也将``Point``类作为:func:`_orm.composite`的第一个参数进行传递::

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

* 直接映射列，将属性名称传递给composite

  我们可以使用更注释的表单来编写上述相同的示例，我们可以选择将属性名称传递给:func:`_orm.composite`
  而不是完整的列构造。

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

  在使用:ref:`imperative table <orm_imperative_table_configuration>`或完全:ref:`imperative <orm_imperative_mapping>`映射时，
  我们直接访问:class:`_schema.Column`对象。这些也可以传递给:func:`_orm.composite`，如下面的命令示例所示::

     mapper_registry.map_imperatively(
         Vertex,
         vertices_table,
         properties={
             "start": composite(Point, vertices_table.c.x1, vertices_table.c.y1),
             "end": composite(Point, vertices_table.c.x2, vertices_table.c.y2),
         },
     )

.. _composite_legacy_no_dataclass:

使用Legacy Non-Dataclasses
----------------------------


如果不使用dataclass，则用户定义的数据类型类的要求是具有构造函数，该构造函数接受与其列格式相对应的位置参数，并且还提供使对象状态作为列表或元组返回的方法``__composite_values __（）''，按其基于列的属性顺序。它还应该提供充分的``__eq__``和``__ne__``方法，用于测试两个实例的相等性。

为了说明主要部分中的等效``Point''类不使用dataclass：

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

使用:func:`~sqlalchemy.orm.composite`的示例继续进行处理，其中还必须使用统一的方法声明关联的列的显式类型，使用:ref:`orm_composite_other_forms` 中的一种形式。

跟踪复合中的原地突变
-----------------------------------------

不会自动跟踪对现有复合值的就地更改。相反，复合类需要明确向其父对象提供事件。通过使用:class:`.MutableComposite` mixin自动完成此任务，该mixin使用事件将每个用户定义的复合对象与所有父关联相关联。有关示例，请参见:ref:`mutable_composites`。

.. _composite_operations:

重新定义复合的比较操作
------------------------------------------------

默认情况下，“equals”比较操作产生所有相应列等于一起的AND。可以使用``comparator_factory''参数将其更改为：func:`~sqlalchemy.orm.composite`，其中我们指定自定义"class:`~sqlalchemy.orm.CompositeProperty.Comparator`类定义现有或新的操作。
下面，我们说明“greater than”运算符，实现与基本“greater than”的相同表达式::

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

由于“Point”是dataclass，因此我们可以使用“dataclasses.astuple（）”来获取“Point”实例的元组形式。

然后定制比较器返回适当的SQL表达式：

.. sourcecode:: pycon+sql

  >>> print(Vertex.start > Point(5, 6))
  {printsql}vertices.x1 > :x1_1 AND vertices.y1 > :y1_1


嵌套复合
-------------------

可以将复合对象定义为在简单的嵌套方案中工作，通过重新定义复合类中的行为以按所需方式工作，然后在通常情况下将复合类映射到单个列的完整长度。这需要定义额外的方法在“嵌套”和“平面”形式之间移动。

以下是我们重新组织“Vertex”类以本身成为引用“Point”对象的composite对象的示例。 "Vertex"和``Point``可以是dataclass，但是我们将向``Vertex''添加一个自定义构造方法，“_generate（）”，以便可以通过将值传递给``Vertex._generate()``
方法来创建新的``Vertex''对象，并将其定义为类方法。

我们还将实现``__composite_values__（）''方法，这是:func:`_orm.composite` 构造函数 （如在前面介绍的:ref:`composite_legacy_no_dataclass`处）的固定名称，指示标准的以列形式接收对象的方式
as a flat tuple of column values，这种情况下将替代通常的面向数据类的方法。

有了我们的自定义“_generate（）”构造函数和“__composite_values__（）”
序列化器方法，我们现在可以在平面tuple和包含Point``对象的``Vertex``对象之间移动。``Vertex._generate``方法作为第一个参数传递给:func:`_orm.composite`，
用于生成新的“Vertex”实例，并且“__composite_values__（）”方法将隐式使用:func:`_orm.composite`。

为了演示用途，将该复合类映射到称为“HasVertex”的类中，其中包含四个源列的:class:`.Table`最终将位于该类别Global:attr:`tables`位置处::

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

上述映射可以按照``HasVertex``，``Vertex``和``Point'的方式使用：的global:attr:'tables`。

    hv = HasVertex(vertex=Vertex(Point(1, 2), Point(3, 4)))

    session.add(hv)
    session.commit()

    stmt = select(HasVertex).where(HasVertex.vertex == Vertex(Point(1, 2), Point(3, 4)))

    hv = session.scalars(stmt).first()
    print(hv.vertex.start)
    print(hv.vertex.end)

.. _dataclass: https://docs.python.org/3/library/dataclasses.html

组合API
-------------

.. autofunction:: composite