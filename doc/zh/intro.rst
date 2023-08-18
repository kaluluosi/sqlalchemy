.. _overview_toplevel:
.. _overview:

========
概述
========

SQLAlchemy SQL工具包和对象关系映射器是一个广泛应用于Python和数据库的工具集。它有几个不同的功能区域，可以单独使用或组合使用。如下图所示，其主要组成部分已经组织成图层，并且组件之间有依赖关系：

.. image:: sqla_arch_small.png

上图展示SQLAlchemy最重要的两个部分应该是**对象关系映射器(ORM)**和**核心(Core)**。

核心包含了SQLAlchemy SQL和数据库集成以及描述服务的大部分内容，其中最突出的部分是**SQL表达式语言(SQL Expression Language)**。

SQL表达式语言本身是一个独立于ORM包的工具包，它提供了一组构建SQL表达式的系统，这些表达式由可组合的对象表示，然后可以在指定事务的范围内的目标数据库上“执行”，并返回结果集。插入、更新和删除(即：term:`DML`)是通过传递SQL表达式对象来实现的，这些语句代表了这些语句的字典，以及要与每个语句一起使用的参数。

ORM基于核心构建了一种工作方式，并与数据库模式映射的域对象模型进行交互。使用ORM，SQL语句的构造方式与使用核心时基本相同，但是DML的任务，这里指的是在数据库中保持业务对象时，使用了称为：term:`unit of work`的模式进行自动化处理，该模式将针对可变对象的状态变化转换为需要插入、更新和删除的构造，然后在这些对象的术语中调用它们。SELECT语句也通过ORM特定的自动化和对象中心查询功能进行增强。

与使用核心和SQL表达式语言的工作方式相比，这提供了关于数据库的基于架构的视图，以及面向不可变性的编程范例。ORM在此基础上构建了基于域的数据库视图，其编程规范更加明确，依赖于可变性。由于关系型数据库本身也是一种可变的服务，因此不同之处在于：Core/SQL表达式语言是面向命令的，而ORM是面向状态的。

.. _doc_overview: 

文档概述
======================

文档分为四个部分：

*   :ref:`unified_tutorial` ——为1.4/2.0系列提供的全新教程，它从Core的描述开始，逐渐向ORM特定的概念等内容展开来介绍整个库。新用户以及从SQLAlchemy 1.x系列过来的用户应该从这里开始。

*   :ref:`orm_toplevel` ——该部分提供ORM的参考文献。

*   :ref:`core_toplevel` ——该部分提供所有Core内部的参考文献。也在这里描述了SQLAlchemy引擎、连接和汇集服务。

*   :ref:`dialect_toplevel` ——提供所有: term:` 方言(dialect)`实现的参考文献，包括  :term:`DBAPI`  细节。

代码示例
=============

SQLAlchemy分发的工作代码示例，主要是关于ORM的。在 :ref:`examples_toplevel` 描述了所有包含的示例应用程序。

在维基上还有大量涉及核心SQLAlchemy结构以及ORM的示例。详见`炼金剧场<https://www.sqlalchemy.org/trac/wiki/UsageRecipes>`_。

.. _installation:

安装指南
==================

支持的平台
-------------------

SQLAlchemy支持以下平台：

* cPython 3.7和更高版本
* Python-3兼容版本的`PyPy <http://pypy.org/>`_

.. versionchanged:: 2.0
    SQLAlchemy现在针对Python 3.7及以上版本。

AsyncIO支持
----------------

SQLAlchemy的``asyncio``支持取决于`greenlet <https://pypi.org/project/greenlet/>`_项目。此依赖项将默认安装在常见的机器平台上，但不支持每种架构，也可能在不常见的体系结构上不会默认安装。请参见   :ref:`asyncio_install` ，了解确保存在asyncio支持的其他详细信息。

支持的安装方法
-------------------------------

SQLAlchemy安装是通过基于`setuptools <https://pypi.org/project/setuptools/>`_的标准Python方法实现的，可以直接引用``setup.py``，也可以使用`pip <https://pypi.org/project/pip/>`_或其他与setuptools兼容的方法。

通过pip安装
---------------

当“pip”可用时，可以通过以下一步下载Python Cheese Shop(PyPI)提供的最新区版本SQLAlchemy的分发和在您的系统上安装它：

.. sourcecode:: text

  pip install SQLAlchemy

这个命令将从Python Cheese Shop(PyPI)下载SQLAlchemy的最新发布版本，并将其安装到您的系统中。对于大多数常见平台，将下载Python Wheel文件，提供了预编译的Cython / C扩展。

为了安装最新的**预发布版**，例如“2.0.0b1”，pip要求要使用“--pre”标志：

.. sourcecode:: text

  pip install --pre SQLAlchemy

上面的命令，如果最新版本是预发布版本，将安装预发布版本而不是最新发布版本。


从源分发手动安装
-------------------------------------------------

未从pip安装时，可以使用“setup.py”脚本安装源分发：

.. sourcecode:: text

  python setup.py install

源代码安装对平台无关，并且将在无论是否安装了Cython/C构建工具的任何平台上进行安装。如下面的章节 :ref:`c_extensions` 所述，“setup.py”将尝试在构建过程中使用Cython/C构建，但是如果失败，则会退回到纯Python安装。

.. _c_extensions:

生成Cython扩展
----------------------------------

SQLAlchemy包括Cython扩展，这些Cython扩展为各个领域提供了额外的速度提升，目前重点是加快核心结果集的速度。

.. versionchanged:: 2.0 C扩展使用Cython重写

如果检测到适当的平台，并且安装了Cython软件包，则“setup.py”将自动构建扩展。完整的手动构建如下所示:

.. sourcecode:: text

  # cd into SQLAlchemy source distribution
  cd path/to/sqlalchemy

  # install cython
  pip install cython

  # optionally build Cython extensions ahead of install
  python setup.py build_ext

  # run the install
  python setup.py install

还可以使用  :pep:`517`  技术，如使用“build_”来执行源构建:

.. sourcecode:: text

  # cd into SQLAlchemy source distribution
  cd path/to/sqlalchemy

  # install build
  pip install build

  # build source / wheel dists
  python -m build

如果Cython扩展的构建失败，由于未安装Cython或编译器缺失等问题，配置过程将输出警告消息，并在完成后不使用Cython扩展重新运行构建，报告最终状态。

要运行构建/安装，而不尝试编译Cython扩展，可以指定“DISABLE_SQLALCHEMY_CEXT”环境变量。这种情况的用例要么是用于特殊测试情况，要么是在通常“重建”机制不能克服的兼容性/构建问题的罕见情况下使用：

.. sourcecode:: text

  export DISABLE_SQLALCHEMY_CEXT=1; python setup.py install


.. _Cython: https://cython.org/

.. _build: https://pypi.org/project/build/


安装数据库API
----------------------------------

SQLAlchemy旨在与为特定数据库构建的: term:`DBAPI`实现一起操作，并包括对最流行数据库的支持。在  :doc:`/dialects/index`  中的各个数据库部分列出了每个数据库的可用DBAPI，包括外部链接。

检查已安装的SQLAlchemy版本
------------------------------------------

本文档涵盖了SQLAlchemy版本2.0。如果您正在处理已经安装了SQLAlchemy的系统，请在Python提示符下检查版本，如下所示：

     >>> import sqlalchemy
     >>> sqlalchemy.__version__  # doctest: +SKIP
     2.0.0

下一步
----------

安装了SQLAlchemy之后，新用户和老用户都可以  :ref:`转到SQLAlchemy教程<unified_tutorial>` 。

.. _migration:

从1.x迁移到2.0
=====================

关于在SQLAlchemy 2.0中发布的新API的说明在这里可以查看  :doc:`changelog/migration_20`  。