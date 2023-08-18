.. _功能_toplevel:
.. _通用函数:

=========================
SQL和通用函数
=========================

.. currentmodule:: sqlalchemy.sql.functions

使用  :data:`_sql.func`  命名空间调用SQL函数。如何使用  :data:` _sql.func`  对象在语句中渲染SQL函数的背景信息请参见 :ref:`tutorial_functions` 中的教程。

.. 参见::

      :ref:`unified_tutorial` 

函数API
------------

SQL函数的基本API，它为  :data:`_sql.func`  命名空间提供类和提供可扩展性的类。

.. autoclass:: AnsiFunction
   :exclude-members: inherit_cache, __new__

.. autoclass:: Function

.. autoclass:: FunctionElement
   :members:
   :exclude-members: inherit_cache, __new__

.. autoclass:: GenericFunction
   :exclude-members: inherit_cache, __new__

.. autofunction:: register_function


选择“已知”函数
--------------------------

这些是基于  :class:`.GenericFunction`  命名空间的任何其他成员以相同的方式调用:：

    select(func.count("*")).select_from(some_table)

请注意，任何未知于  :data:`_sql.func`  的名称都会按照原样生成函数名称——没有限制可以调用什么SQL函数，无论是SQLAlchemy所知，还是内置或用户定义的。本节仅描述SQLAlchemy已经知道何种参数和返回类型的函数。

.. autoclass:: array_agg
    :no-members:

.. autoclass:: char_length
    :no-members:

.. autoclass:: coalesce
    :no-members:

.. autoclass:: concat
    :no-members:

.. autoclass:: count
    :no-members:

.. autoclass:: cube
    :no-members:

.. autoclass:: cume_dist
    :no-members:

.. autoclass:: current_date
    :no-members:

.. autoclass:: current_time
    :no-members:

.. autoclass:: current_timestamp
    :no-members:

.. autoclass:: current_user
    :no-members:

.. autoclass:: dense_rank
    :no-members:

.. autoclass:: grouping_sets
    :no-members:

.. autoclass:: localtime
    :no-members:

.. autoclass:: localtimestamp
    :no-members:

.. autoclass:: max
    :no-members:

.. autoclass:: min
    :no-members:

.. autoclass:: mode
    :no-members:

.. autoclass:: next_value
    :no-members:

.. autoclass:: now
    :no-members:

.. autoclass:: percent_rank
    :no-members:

.. autoclass:: percentile_cont
    :no-members:

.. autoclass:: percentile_disc
    :no-members:

.. autoclass:: random
    :no-members:

.. autoclass:: rank
    :no-members:

.. autoclass:: rollup
    :no-members:

.. autoclass:: session_user
    :no-members:

.. autoclass:: sum
    :no-members:

.. autoclass:: sysdate
    :no-members:

.. autoclass:: user
    :no-members: