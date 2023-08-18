游客和遍历工具
===============

  :mod:`sqlalchemy.sql.visitors`   模块由类和函数组成，旨在通用地**遍历**核心 SQL 表达式结构，这类似于 Python 中的 ` `ast`` 模块，因为它提供了一个程序可以操作 SQL 表达式的每个组件的系统。它常用于查找各种类型的元素，例如  :class:`_schema.Table` .BindParameter` 对象，以及更改该结构的状态，例如用其他 FROM 子句替换某些子句。

.. note::  :mod:`sqlalchemy.sql.visitors`  模块是一个内部 API，不是完全公开的。它可能会被更改，并且可能不像预期的那样适用于 SQLAlchemy 内部未考虑到的使用模式。

  :mod:`sqlalchemy.sql.visitors`   模块是 SQLAlchemy 内部的一部分，通常情况下不会由调用应用程序代码使用。但是，在某些边缘情况下会使用它，例如构建缓存例程时以及使用   :ref:` Custom SQL Constructs and Compilation Extension <sqlalchemy.ext.compiler_toplevel>`  构建自定义 SQL 表达式时。

.. automodule:: sqlalchemy.sql.visitors
   :members:
   :private-members: