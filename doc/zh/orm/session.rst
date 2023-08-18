.. _session_toplevel:

用 Session
==========

.. module:: sqlalchemy.orm.session

在ORM中，   :ref:`mapper_config_toplevel`  描述的映射函数声明和ORM是最主要的配置界面。一旦配置好映射，持久性操作的主要使用界面就是   :class:` .Session` .

.. toctree::
    :maxdepth: 3

    session_basics
    session_state_management
    cascades           # ** 多连表相关，译者注 **
    session_transaction
    persistence_techniques
    contextual
    session_events
    session_api