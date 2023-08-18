.. _plugins:
.. _sqlalchemy.ext:

ORM扩展
========

SQLAlchemy有很多ORM扩展可用，它们为核心行为添加了额外的功能。

扩展几乎完全基于公共的核心和ORM API，并鼓励用户阅读其源代码以进一步了解其行为。特别是“水平分片”、“混合属性”和“突变跟踪”扩展非常简洁。

.. toctree::
    :maxdepth: 1

    asyncio
    associationproxy
    automap
    baked
    declarative/index
    mypy
    mutable
    orderinglist
    horizontal_shard
    hybrid
    indexable
    instrumentation