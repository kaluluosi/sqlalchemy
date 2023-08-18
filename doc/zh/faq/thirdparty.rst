第三方集成问题
=======================

.. contents::
    :local:
     :class: faq
    :backlinks: none

.. _numpy_int64:

我收到了与“ ``numpy.int64`` ”，“``numpy.bool_`` ”等相关的错误。
-------------------------------------------------- ----------------------

numpy_包具有自己的数字数据类型，这些数据类型从Python的数字类型扩展，但包含某些行为，有些情况下使它们无法与SqlAlchemy的某些行为相协调，有些情况下与底层的DBAPI驱动程序不兼容。

可能会发生两个错误是``ProgrammingError：cannot adapt type 'numpy.int64'``等后端出现错误，以及``ArgumentError：SQL expression object expected，got object of type <class 'numpy.bool_'> instead``; 在SqlAlchemy的更高版本中，这可能是``ArgumentError：SQL expression for WHERE / HAVING role expected，got True``。

在第一种情况下，问题是因为psycopg2没有适当的查找条目以使 ``int64`` 数据类型不被直接接受。这可以基于以下代码说明：

    import numpy


    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        data = Column(Integer)


    # .. later
    session.add(A(data=numpy.int64(10)))
    session.commit()

在后一种情况下，问题是因为 ``numpy.int64`` 数据类型覆盖了 ``__eq __（）`` 方法，并强制执行表达式的返回类型为 ``numpy.True`` 或 ``numpy.False``，这会破坏SQLAlchemy的表达式语言行为，因为表达式语言预计从Python等式比较中返回：class:`~sqlalchemy.sql.expression.ColumnElement`表达式：

.. sourcecode:: pycon+sql

    >>> import numpy
    >>> from sqlalchemy import column, Integer
    >>> print(column("x", Integer) == numpy.int64(10))  # works
    {printsql}x = :x_1{stop}
    >>> print(numpy.int64(10) == column("x", Integer))  # breaks
    False

这些错误都可以通过相同的方式解决，即特殊的numpy数据类型需要用普通的Python值替换。例如，将Python的``int()``
函数应用于像 ``numpy.int32`` 和 ``numpy.int64`` 这样的类型，将 Python ``float()`` 函数应用于``numpy.float32``：

    data = numpy.int64(10)

    session.add(A(data=int(data)))

    result = session.execute(select(A.data).where(int(data) == A.data))

    session.commit()

.. _numpy: https://numpy.org

WHERE/HAVING角色的SQL表达式期望得到True
-----------------------------------------------------

请参阅：  :ref:`numpy_int64` .