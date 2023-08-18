SELECT和相关结构
==========================

“可选择（selectable）”一词是指表示数据库行的任何对象。在SQLAlchemy中，这些对象都继承自:class:`_expression.Selectable`，其中最突出的是:class:`_expression.Select`，它代表了一个SQL SELECT语句。 :class:`_expression.FromClause`是:class:`_expression.Selectable`的一个子集，它代表可以在:class:`.Select`语句的FROM子句中包含的对象。 :class:`_expression.FromClause`的一个区别特征是 :attr:`_expression.FromClause.c` 属性，它是FROM子句中包含的所有列的命名空间（这些元素本身是 :class:`_expression.ColumnElement`的子类）。

.. currentmodule:: sqlalchemy.sql.expression

.. _selectable_foundational_constructors:

可选择的基础构造函数
--------------------------------------

顶级“FROM子句”和“SELECT”构造函数。


.. autofunction:: except_

.. autofunction:: except_all

.. autofunction:: exists

.. autofunction:: intersect

.. autofunction:: intersect_all

.. autofunction:: select

.. autofunction:: table

.. autofunction:: union

.. autofunction:: union_all

.. autofunction:: values


.. _fromclause_modifier_constructors:

可选择的修饰符构造函数
---------------------------------

此处列出的函数更常见地作为:class:`_sql.FromClause`和:class:`_sql.Selectable`元素的方法可用，例如，:func:`_sql.alias`函数通常通过 :meth:`_sql.FromClause.alias`方法调用。

.. autofunction:: alias

.. autofunction:: cte

.. autofunction:: join

.. autofunction:: lateral

.. autofunction:: outerjoin

.. autofunction:: tablesample


可选择的类文档
--------------------------------

以下类是使用列在 :ref:`selectable_foundational_constructors` 和 :ref:`fromclause_modifier_constructors` 中列出的构造函数生成的。

.. autoclass:: Alias
   :members:

.. autoclass:: AliasedReturnsRows
   :members:

.. autoclass:: CompoundSelect
   :inherited-members: ClauseElement
   :members:

.. autoclass:: CTE
   :members:

.. autoclass:: Executable
   :members:

.. autoclass:: Exists
   :members:

.. autoclass:: FromClause
   :members:

.. autoclass:: GenerativeSelect
   :members:

.. autoclass:: HasCTE
   :members:

.. autoclass:: HasPrefixes
   :members:

.. autoclass:: HasSuffixes
   :members:

.. autoclass:: Join
   :members:

.. autoclass:: Lateral
   :members:

.. autoclass:: ReturnsRows
   :members:
   :inherited-members: ClauseElement

.. autoclass:: ScalarSelect
   :members:

.. autoclass:: Select
   :members:
   :inherited-members: ClauseElement
   :exclude-members: memoized_attribute, memoized_instancemethod, append_correlation, append_column, append_prefix, append_whereclause, append_having, append_from, append_order_by, append_group_by


.. autoclass:: Selectable
   :members:
   :inherited-members: ClauseElement

.. autoclass:: SelectBase
   :members:
   :inherited-members: ClauseElement
   :exclude-members: memoized_attribute, memoized_instancemethod

.. autoclass:: Subquery
   :members:

.. autoclass:: TableClause
   :members:
   :inherited-members:

.. autoclass:: TableSample
   :members:

.. autoclass:: TableValuedAlias
   :members:

.. autoclass:: TextualSelect
   :members:
   :inherited-members:

.. autoclass:: Values
   :members:

.. autoclass:: ScalarValues
   :members:



标签样式常量
---------------------

在:meth:`_sql.GenerativeSelect.set_label_style`方法中使用的常量。

.. autoclass:: SelectLabelStyle
    :members:


.. seealso::

    :meth:`_sql.Select.set_label_style`

    :meth:`_sql.Select.get_label_style`