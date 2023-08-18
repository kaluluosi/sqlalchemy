.. _orm_event_toplevel:

ORM 事件
==========

ORM 包括一系列可供订阅的钩子。

关于最常用的 ORM 事件的介绍，请参见章节   :ref:`session_events_toplevel` 。一般的事件系统在章节   :ref:` event_toplevel`  中介绍，关于连接和低级语句执行等非 ORM 事件在章节   :ref:`core_event_toplevel`  中描述。

会话事件
---------------

最基本的事件钩子可在 ORM   :class:`_orm.Session`  对象级别获得。可以截获的操作包括：

* **持久性操作** - ORM 冲刷将更改发送到数据库的过程可以使用在冲刷的不同部分发出的事件进行扩展，以增强或修改发送到数据库的数据或允许其他事情在持久性发生时发生。有关持久性事件的详细信息，请参阅章节   :ref:`session_persistence_events` 。

* **对象生命周期事件** - 当对象添加到会话中、持久化或从会话中删除时发出的钩子。有关这些的详细信息，请参见章节   :ref:`session_lifecycle_events` 。

* **执行事件** - 在  :term:`2.0 style`  执行模型的一部分中，对于针对 ORM 实体发出的所有 SELECT 语句以及冲刷过程之外的批量 UPDATE 和 DELETE 语句，可以使用  :meth:` _orm.SessionEvents.do_orm_execute`  方法从  :meth:`_orm.Session.execute`  方法截取操作。有关此事件的详细信息，请参见章节   :ref:` session_execute_events` 。

请确保阅读   :ref:`session_events_toplevel`  章节以获取上下文的内容。

.. autoclass:: sqlalchemy.orm.SessionEvents
   :members:

映射器事件
-------------

映射器事件钩子涵盖与单个或多个   :class:`_orm.Mapper`  对象相对应的事物，这些对象是将用户定义的类映射到   :class:` _schema.Table`  对象的中心配置对象。在   :class:`_orm.Mapper`  级别发生的事物的类型包括：

* **每个对象的持久化操作** - 最受欢迎的映射器钩子通常是诸如  :meth:`_orm.MapperEvents.before_insert` 、  :meth:` _orm.MapperEvents.after_update`   等的工作单元钩子。与在管道过程中发生的较粗粒度会话级事件相比，这些事件在对于每个对象在冲刷过程中发生，虽然对象上的更细粒度的活动更加简单，但   :class:`_orm.Session`  功能的可用性有限。

* **映射器配置事件** - 映射器钩子的另一个主要类别是在映射类时发生的事物，以及在映射器完成时，将映射器配置为相互引用的映射器集合时发生的事物。这些事件包括  :meth:`_orm.MapperEvents.instrument_class` 、  :meth:` _orm.MapperEvents.before_mapper_configured`   和  :meth:`_orm.MapperEvents.mapper_configured`  在单个   :class:` _orm.Mapper`  级别，以及  :meth:`_orm.MapperEvents.before_configured`  和  :meth:` _orm.MapperEvents.after_configured`  在一组   :class:`_orm.Mapper`  对象级别。

.. autoclass:: sqlalchemy.orm.MapperEvents
   :members:

实例事件
---------------

实例事件侧重于 ORM 映射实例的构造，包括在实例化为  :term:`transient`  对象时、从数据库加载并变为  :term:` persistent`  对象时，以及在对象上发生数据库刷新或过期操作时。

.. autoclass:: sqlalchemy.orm.InstanceEvents
   :members:


.. _orm_attribute_events:

属性事件
----------------

属性事件是针对 ORM 映射对象的各个属性发生的事件。这些事件构成了诸如   :ref:`custom validation functions <simple_validators>`  和   :ref:` backref handlers <relationships_backref>`  等的基础。

.. seealso::

    :ref:`mapping_attributes_toplevel` 

.. autoclass:: sqlalchemy.orm.AttributeEvents
   :members:


查询事件
------------

.. autoclass:: sqlalchemy.orm.QueryEvents
   :members:

仪器事件
----------------------

.. automodule:: sqlalchemy.orm.instrumentation

.. autoclass:: sqlalchemy.orm.InstrumentationEvents
   :members: