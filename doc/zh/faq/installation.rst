安装
=================

.. contents::
    :local:
    :class: faq
    :backlinks: none

.. _faq_asyncio_installation:

当我尝试使用asyncio时，出现了关于greenlet未安装的错误
----------------------------------------------------------------------------------

``greenlet`` 依赖在对于 ``greenlet`` 没有提供 `预编译二进制包 <https://pypi.org/project/greenlet/#files>`_ 的CPU架构默认没有安装。
特别地，**这包括了Apple M1**。 为了安装包含 ``greenlet`` 的版本，请在 ``pip install`` 命令中添加 ``asyncio`` 的 `setuptools额外项 <https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras>`_：

.. sourcecode:: text

    pip install sqlalchemy[asyncio]

更多详细信息，请参阅 :ref:`asyncio_install`。


.. seealso::

    :ref:`asyncio_install`