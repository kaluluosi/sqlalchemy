选择和相关结构
=================

“selectable”这个术语指代所有代表数据库行的对象。在SQLAlchemy中，这些对象从   :class:`_expression.Selectable`  继承，其中最显著的是   :class:` _expression.Select` ，表示 SQL SELECT 语句。   :class:`_expression.Selectable`  的一个子集是   :class:` _expression.FromClause` ，表示可以在   :class:`.Select`  语句的 FROM 子句中的对象。   :class:` _expression.FromClause`  的一个显著特征是  :attr:`_expression.FromClause.c`  属性，它是 FROM 子句中包含的所有列的名称空间（这些元素本身是   :class:` _expression.ColumnElement`  子类）。

.. currentmodule:: sqlalchemy.sql.expression

.. _selectable_foundational_constructors:

可选择的基础构造器
--------------------------

顶级“FROM子句”和“SELECT”构造器。

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

可选择的修饰符构造器
---------------------------------

此处列出的函数通常作为   :class:`_sql.FromClause`  和   :class:` _sql.Selectable`  元素的方法更常见，例如，   :func:`_sql.alias`  函数通常通过  :meth:` _sql.FromClause.alias`  方法调用。

.. autofunction:: alias

.. autofunction:: cte

.. autofunction:: join

.. autofunction:: lateral

.. autofunction:: outerjoin

.. autofunction:: tablesample


Selectable类文档
--------------------------------

这里的类是使用   :ref:`selectable_foundational_constructors`  和   :ref:` fromclause_modifier_constructors`  中列出的构造器生成的。

.. autoclass:: Alias
   :members:

.. autoclass:: AliasedReturnsRows
   :members:

.. autoclass:: CompoundSelect
   :inherited-members:  ClauseElement
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
   :inherited-members:  ClauseElement

.. autoclass:: ScalarSelect
   :members:

.. autoclass:: Select
   :members:
   :inherited-members:  ClauseElement
   :exclude-members: memoized_attribute, memoized_instancemethod, append_correlation, append_column, append_prefix, append_whereclause, append_having, append_from, append_order_by, append_group_by


.. autoclass:: Selectable
   :members:
   :inherited-members: ClauseElement

.. autoclass:: SelectBase
   :members:
   :inherited-members:  ClauseElement
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

  :meth:`_sql.GenerativeSelect.set_label_style`   方法使用的常量。

.. autoclass:: SelectLabelStyle
    :members:


.. seealso::

     :meth:`_sql.Select.set_label_style` 

     :meth:`_sql.Select.get_label_style` 