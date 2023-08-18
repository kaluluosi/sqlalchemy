安装
=================

.. contents::
    :local:
     :class: faq
    :backlinks: none

.. _faq_asyncio_installation:

当我尝试使用asyncio时，出现了greenlet未安装的错误
----------------------------------------------------------------------------------

默认情况下，对于``greenlet``没有提供' `预构建的二进制wheel文件<https://pypi.org/project/greenlet/#files>`_
用于CPU架构不提供该文件的情况下，其不会安装。
尤其是，**这包括苹果 M1**。要包括``greenlet``安装，需向``pip install``添加``asyncio``
`setuptools extra<https://packaging.python.org/en/latest/tutorials/installing-packages/#installing-setuptools-extras>`_ ：

.. sourcecode:: text

    pip install sqlalchemy[asyncio]

有关更多背景信息，请参见：   :ref:`asyncio_install`  。


.. seealso::

      :ref:`asyncio_install` 