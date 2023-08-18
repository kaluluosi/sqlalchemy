.. _session_toplevel:

=================
使用Session
=================

.. module:: sqlalchemy.orm.session

在ORM中，上一节描述的declarative base和ORM映射函数是主要的配置界面。一旦映射配置完成，持久化操作的主使用接口为:class:`.Session`。

.. toctree::
    :maxdepth: 3

    session_basics
    session_state_management
    cascades
    session_transaction
    persistence_techniques
    contextual
    session_events
    session_api