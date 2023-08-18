.. _plugins:
.. _sqlalchemy.ext:

ORM扩展
=======

SQLAlchemy提供了各种ORM扩展，其添加了基本行为之外的额外功能。

这些扩展几乎完全基于公共核心和ORM API，建议用户阅读其源代码以进一步了解其行为。特别是"横向分片"，"混合属性"和"突变跟踪"扩展非常简洁。

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