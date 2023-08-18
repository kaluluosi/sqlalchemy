会话API
===========

会话和sessionmaker()
--------------------------

.. autoclass:: sessionmaker
    :members:
    :inherited-members:

.. autoclass:: ORMExecuteState
    :members:

.. autoclass:: Session
   :members:
   :inherited-members:

.. autoclass:: SessionTransaction
   :members:

.. autoclass:: SessionTransactionOrigin
   :members:

会话工具
-----------------

.. autofunction:: close_all_sessions

.. autofunction:: make_transient

.. autofunction:: make_transient_to_detached

.. autofunction:: object_session

.. autofunction:: sqlalchemy.orm.util.was_deleted

属性和状态管理工具
----------------------------------------

这些函数由SQLAlchemy属性工具API提供，以提供一个详细的接口来处理实例、属性值和历史记录。当构建事件监听器函数时，其中一些函数是有用的，例如在 :doc:`/orm/events` 中描述的函数。

.. currentmodule:: sqlalchemy.orm.util

.. autofunction:: object_state

.. currentmodule:: sqlalchemy.orm.attributes

.. autofunction:: del_attribute

.. autofunction:: get_attribute

.. autofunction:: get_history

.. autofunction:: init_collection

.. autofunction:: flag_modified

.. autofunction:: flag_dirty

.. function:: instance_state

    返回给定映射对象的   :class:`.InstanceState` 。

    此函数是   :func:`.object_state`  的内部版本。
    这里建议使用 `  :func:`.object_state`  和/或
    `  :func:`_sa.inspect`  函数，因为如果给定的对象未映射，它们都会发出一个可读的异常。

.. autofunction:: sqlalchemy.orm.instrumentation.is_instrumented

.. autofunction:: set_attribute

.. autofunction:: set_committed_value

.. autoclass:: History
    :members: