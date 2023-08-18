第三方集成问题
===============================

.. contents::
    :local:
    :class: faq
    :backlinks: none

.. _numpy_int64:

我得到了与“``numpy.int64``”，“``numpy.bool_``”等相关的错误。
------------------------------------------------------------------------

numpy_包有其自己的数字数据类型，这些数据类型扩展了Python的数字类型，但它们包含某些行为，这些行为在某些情况下使它们无法与SQLAlchemy的一些行为协调，以及在某些情况下 用底层的DBAPI驱动程序不兼容。

可能会出现两个错误：“ProgrammingError：无法适应类型``numpy.int64''”在像psycopg2这样的后端上，以及“ArgumentError：SQL表达式对象预期得到的是<class 'numpy.bool_'>类型的对象，而不是实际情况”的情况 在最近的SQLAlchemy版本中，可能是“ArgumentError：SQL表达式为WHERE / HAVING预期的角色，得到True”。

在第一种情况下，问题是由于psycopg2没有适当的查找项来查找“int64”数据类型，因此它不会被查询直接接受。这可以基于以下代码进行说明：

    import numpy

    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        data = Column(Integer)

    # .. 后面
    session.add(A(data=numpy.int64(10)))
    session.commit()

在后一种情况下，问题是由于``numpy.int64``数据类型覆盖了``__eq __（）``方法并强制执行表达式的返回类型为``numpy.True''或``numpy.False''，这将破坏SQLAlchemy的表达式功能语法，该语法希望从python相等比较返回:class：``_sql.ColumnElement``表达式：

.. sourcecode:: pycon+sql

    >>> import numpy
    >>> from sqlalchemy import column, Integer
    >>> print(column("x", Integer) == numpy.int64(10))  # works
    {printsql}x = :x_1{stop}
    >>> print(numpy.int64(10) == column("x", Integer))  # breaks
    False

这些错误都可以通过相同的方式解决，即需要将特殊的numpy数据类型替换为常规的Python值。例如，将Python ``int（）``函数应用于诸如``numpy.int32``和``numpy.int64``之类的类型，以及将Python ``float（）``函数应用于``numpy.float32``：

    data = numpy.int64(10)

    session.add(A(data=int(data)))

    result = session.execute(select(A.data).where(int(data) == A.data))

    session.commit()

.. _numpy: https://numpy.org

SQL表达式为WHERE/HAVING预期的角色，得到True
-------------------------------------------------------

请参见 :ref:`numpy_int64`.