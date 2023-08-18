列元素和表达式
===============================

.. currentmodule:: sqlalchemy.sql.expression

表达式API由一系列表示SQL字符串中特定词法元素的类组成。当组合在一起形成一个语句构造时，它们可以被编译成一个字符串表示形式，可以传递给一个数据库。这些类组织成一个层次结构，从最基本的:class:`.ClauseElement`类开始。重要的子类包括 :class:`.ColumnElement`，表示SQL语句中任何基于列的表达式的角色，例如在列子句、WHERE子句和ORDER BY子句中，以及:class:`.FromClause`，表示在SELECT语句的FROM子句中放置的标记的角色。

.. _sqlelement_foundational_constructors:

列元素初始构造函数
-----------------------------------------

从``sqlalchemy``名称空间导入的独立函数，用于构建SQLAlchemy表达式语言构造时使用。

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

      字符串是否应无条件引用


.. _sqlelement_modifier_constructors:

列元素修饰符构造函数
-------------------------------------

此处列出的功能更常见作为从任何:class:` _sql.ColumnElement`构造的方法使用，例如，:func:`_sql.label`函数通常通过:meth:` _sql.ColumnElement.label`方法调用。

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

   :func:`_sql.nulls_first`函数的同义词。

   .. versionchanged:: 2.0.5 恢复缺少的遗留符号 :func:`.nullsfirst`。

.. autofunction:: nulls_last

.. function:: nullslast

   :func:`_sql.nulls_last`函数的遗留同义词。

   .. versionchanged:: 2.0.5 恢复缺少的遗留符号 :func:`.nullslast`。

.. autofunction:: over

.. autofunction:: within_group

列元素类文档
-----------------------------------

这里列出的类是使用在 :ref:`sqlelement_foundational_constructors`和
:ref:`sqlelement_modifier_constructors` 列出的构造函数生成的。

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

   General purpose "column expression" argument.

   .. versionadded:: 2.0.13

   这种类型用于表示单个SQL列表达式的"列"表达式，包括
   :class:`_sql.ColumnElement`，以及ORM映射的属性，这些属性将有一个``__clause_element__()``
   方法。


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