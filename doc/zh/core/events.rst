.. _core_event_toplevel:

核心事件
===========

本节描述SQLAlchemy Core提供的事件接口。
有关事件侦听API的介绍，请参见：:ref:`event_toplevel`。
ORM事件请参见：:ref:`orm_event_toplevel`。

.. autoclass:: sqlalchemy.event.base.Events
   :members:

连接池事件
----------------------

.. autoclass:: sqlalchemy.events.PoolEvents
   :members:

.. autoclass:: sqlalchemy.events.PoolResetState
   :members:

.. _core_sql_events:

SQL执行和连接事件
-----------------------------------

.. autoclass:: sqlalchemy.events.ConnectionEvents
    :members:

.. autoclass:: sqlalchemy.events.DialectEvents
    :members:

架构事件
-------------

.. autoclass:: sqlalchemy.events.DDLEvents
    :members:

.. autoclass:: sqlalchemy.events.SchemaEventTarget
    :members: