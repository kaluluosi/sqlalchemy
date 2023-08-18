列元素和表达式
=============

.. currentmodule:: sqlalchemy.sql.expression

表达式API由一系列类组成，每个类代表SQL字符串中的一个特定的词素元素。它们组成一个查询构造，可以*编译*成一个字符串表示形式，然后可以传递到数据库里。这些类组织成一个基类最底层的   :class:`.ClauseElement`  类的层次结构。关键子类包括  :class:` .ColumnElement` ，表示SQL语句中任何基于列的表达式的角色，例如在列、WHERE子句和ORDER BY子句中，以及  :class:`.FromClause` ，表示放置在SELECT语句的FROM子句中的token的作用。

.. _sqlelement_foundational_constructors:

列元素基础构造函数
-----------------------

从“sqlalchemy”命名空间导入的独立函数，用于构建SQLAlchemy表达式语言构造。

.. autofunction:: and_

.. autofunction:: bindparam

.. autofunction:: bitwise_not

.. autofunction:: case

.. autofunction:: cast

.. autofunction:: column

.. autoclass:: custom_op
   :members:

.. autofunction:: distinct

.. autofunction:: extract

.. autofunction:: false

.. autodata:: func

.. autofunction:: lambda_stmt

.. autofunction:: literal

.. autofunction:: literal_column

.. autofunction:: not_

.. autofunction:: null

.. autofunction:: or_

.. autofunction:: outparam

.. autofunction:: text

.. autofunction:: true

.. autofunction:: try_cast

.. autofunction:: tuple_

.. autofunction:: type_coerce

.. autoclass:: quoted_name

   .. attribute:: quote

      字符串是否应该被无条件引用

.. _sqlelement_modifier_constructors:

Column Element Modifier Constructors
-------------------------------------

此处列出的函数通常作为任何   :class:`_sql.ColumnElement`  构造的方法更常见，例如，  :func:` _sql.label`  函数通常通过  :meth:`_sql.ColumnElement.label`  方法调用。

.. autofunction:: all_

.. autofunction:: any_

.. autofunction:: asc

.. autofunction:: between

.. autofunction:: collate

.. autofunction:: desc

.. autofunction:: funcfilter

.. autofunction:: label

.. autofunction:: nulls_first

.. function:: nullsfirst

    :func:`_sql.nulls_first`  函数的同义词。

  .. versionchanged:: 2.0.5 计划恢复缺失的遗留符号   :func:`.nullsfirst` 。

.. autofunction:: nulls_last

.. function:: nullslast

    :func:`_sql.nulls_last`  函数的遗留同义词。

  .. versionchanged:: 2.0.5 计划恢复缺失的遗留符号   :func:`.nullslast` 。

.. autofunction:: over

.. autofunction:: within_group

列元素类文档
-------------------

这里的类使用在   :ref:`sqlelement_foundational_constructors`  和   :ref:` sqlelement_modifier_constructors`  中列出的构造函数创建。

.. autoclass:: BinaryExpression
   :members:

.. autoclass:: BindParameter
   :members:

.. autoclass:: Case
   :members:

.. autoclass:: Cast
   :members:

.. autoclass:: ClauseList
   :members:

.. autoclass:: ColumnClause
   :members:

.. autoclass:: ColumnCollection
   :members:

.. autoclass:: ColumnElement
   :members:
   :inherited-members:
   :undoc-members:

.. data:: ColumnExpressionArgument

   通用的“列表达式”参数。

   .. versionadded:: 2.0.13

   此类型用于表示"列"类型的表达式，通常代表单个SQL列表达式，包括   :class:`_sql.ColumnElement` ，以及将具有一个 ` `__clause_element__()`` 方法的ORM映射属性。

.. autoclass:: ColumnOperators
   :members:
   :special-members:
   :inherited-members:

.. autoclass:: Extract
   :members:

.. autoclass:: False_
   :members:

.. autoclass:: FunctionFilter
   :members:

.. autoclass:: Label
   :members:

.. autoclass:: Null
   :members:

.. autoclass:: Operators
   :members:
   :special-members:

.. autoclass:: Over
   :members:

.. autoclass:: SQLColumnExpression

.. autoclass:: TextClause
   :members:

.. autoclass:: TryCast
   :members:

.. autoclass:: Tuple
   :members:

.. autoclass:: WithinGroup
   :members:

.. autoclass:: sqlalchemy.sql.elements.WrapsColumnExpression
   :members:

.. autoclass:: True_
   :members:

.. autoclass:: TypeCoerce
   :members:

.. autoclass:: UnaryExpression
   :members: