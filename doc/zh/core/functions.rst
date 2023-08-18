.. _functions_toplevel:
.. _generic_functions:

=========================
SQL和通用函数
=========================

.. currentmodule:: sqlalchemy.sql.functions

SQL函数可通过使用 :data:`_sql.func` 命名空间来调用。有关如何在语句中使用 :data:`_sql.func` 对象渲染 SQL 函数的背景信息，请参见 :ref:`tutorial_functions` 中的教程。

.. seealso::

    :ref:`unified_tutorial` 中的 :ref:`tutorial_functions`

函数 API
------------

SQL函数的基本 API，提供 :data:`_sql.func` 命名空间和可用于可扩展性的类。

.. autoclass:: AnsiFunction
   :exclude-members: inherit_cache, __new__

.. autoclass:: Function

.. autoclass:: FunctionElement
   :members:
   :exclude-members: inherit_cache, __new__

.. autoclass:: GenericFunction
   :exclude-members: inherit_cache, __new__

.. autofunction:: register_function


"已知"函数
--------------------------

以下是某个常见 SQL 函数的 :class:`.GenericFunction` 实现，自动设置每个函数的预期返回类型。它们的调用方式与 :data:`_sql.func` 命名空间的任何其他成员相同::

    select(func.count("*")).select_from(some_table)

请注意，任何名称未知于 :data:`_sql.func` 的内容都会生成函数名称 - 没有限制，可以调用任何 SQL 函数，无论是 SQLAlchemy 知道还是不知道，内置的或用户定义的。本节仅描述 SQLAlchemy 已经知道使用了什么参数和返回类型的那些函数。

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