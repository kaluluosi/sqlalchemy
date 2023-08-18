访问者和遍历工具
==================

:mod:`sqlalchemy.sql.visitors` 模块由类和函数组成，旨在通用地**遍历**核心 SQL 表达式结构。这类似于 Python ``ast`` 模块，因为它提供了一种程序可以处理 SQL 表达式每个组件的系统。它常见的目的是定位各种元素，如 :class:`_schema.Table` 或 :class:`.BindParameter` 对象，并更改结构的状态，例如将某些 FROM 子句替换为其他子句。

.. note:: :mod:`sqlalchemy.sql.visitors` 模块是一个内部 API，并非完全公开。它可能会发生变化，并且可能不会按照 SQLAlchemy 中未考虑的使用模式的预期方式运行。

:mod:`sqlalchemy.sql.visitors` 模块是 SQLAlchemy 的**内部**模块，通常不会被调用应用程序代码使用。然而，在某些边缘情况下，例如构建缓存例程以及使用 :ref:`自定义 SQL 构造和编译扩展 <sqlalchemy.ext.compiler_toplevel>` 构建自定义 SQL 表达式时，该模块还是有用的。

.. automodule:: sqlalchemy.sql.visitors
   :members:
   :private-members: