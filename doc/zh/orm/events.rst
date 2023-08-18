.. _orm_event_toplevel:

ORM 事件
========

ORM 包含了许多可订阅的钩子。

关于最常用的 ORM 事件的介绍，请参见章节 :ref:`session_events_toplevel`。通常的事件系统在章节 :ref:`event_toplevel` 中讨论。描述连接和低级语句执行等非 ORM 事件的文档请参见章节 :ref:`core_event_toplevel`。

会话事件
--------

最基本的事件钩子可在 ORM 的 :class:`_orm.Session` 对象级别上使用。此处可截获的内容包括：

* **持久化操作** - ORM flush 过程可使用以不同方式触发的事件进行扩展，以增强或修改发送到数据库的数据，或在持久性发生时允许发生其他事情。详见章节 :ref:`session_persistence_events`。

* **对象生命周期事件** - 对象添加、持久化、从会话中删除等事件的钩子。详见章节 :ref:`session_lifecycle_events`。

* **查询事件** - 在 :term:`2.0 style` 执行模式的一部分中，对 ORM 实体执行的所有 SELECT 语句以及批量的 UPDATE 和 DELETE 语句（不在 flush 过程中）将从 :meth:`_orm.Session.execute` 方法中截获。使用 :meth:`_orm.SessionEvents.do_orm_execute` 方法。详见章节 :ref:`session_execute_events` 中的这个事件。

请务必阅读 :ref:`session_events_toplevel` 章节以了解这些事件的背景信息。

.. autoclass:: sqlalchemy.orm.SessionEvents
   :members:

映射器事件
----------

映射器事件包括与一个或多个 :class:`_orm.Mapper` 对象相关的事件，这是将用户定义的类映射到 :class:`_schema.Table` 对象的中心配置对象。在 :class:`_orm.Mapper` 级别上发生的类型包括：

* **每个对象的持久化操作** - 最常用的 Mapper 钩子是像 :meth:`_orm.MapperEvents.before_insert`、:meth:`_orm.MapperEvents.after_update` 等的工作单元钩子。这些事件与更粗粒度的会话级事件（如 :meth:`_orm.SessionEvents.before_flush`）不同，因为它们在刷新过程中以每个对象为基础发生。虽然对象上的更细粒度的活动更为直接，但可用的 :class:`_orm.Session` 特性有限。

* **映射器配置事件** - 另一个重要的映射器钩子类是在将类映射时、映射器完成时以及将映射器集配置为相互引用时发生的钩子。这些事件包括 :meth:`_orm.MapperEvents.instrument_class`、:meth:`_orm.MapperEvents.before_mapper_configured` 和 :meth:`_orm.MapperEvents.mapper_configured` 等这些事件在每个 :class:`_orm.Mapper` 的级别，以及 :meth:`_orm.MapperEvents.before_configured` 和 :meth:`_orm.MapperEvents.after_configured` 在某些集合级别的 :class:`_orm.Mapper` 对象中。

.. autoclass:: sqlalchemy.orm.MapperEvents
   :members:

实例事件
--------

实例事件以 ORM 映射实例的构建为中心，包括在将它们实例化为 :term:`transient` 对象时、在它们从数据库中加载并变为 :term:`persistent` 对象时，以及当在对象上进行数据库刷新或失效操作时发生的事件。

.. autoclass:: sqlalchemy.orm.InstanceEvents
   :members:



.. _orm_attribute_events:

属性事件
--------

属性事件在 ORM 映射对象的个别属性上触发，这些事件形成了像 :ref:`简单验证函数 <simple_validators>` 和 :ref:`反向引用处理程序 <relationships_backref>` 这样的基础。

.. seealso::

  :ref:`mapping_attributes_toplevel`

.. autoclass:: sqlalchemy.orm.AttributeEvents
   :members:


查询事件
--------

.. autoclass:: sqlalchemy.orm.QueryEvents
   :members:

仪表盘事件
----------

.. automodule:: sqlalchemy.orm.instrumentation

.. autoclass:: sqlalchemy.orm.InstrumentationEvents
   :members: