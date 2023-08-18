.. _event_toplevel:

事件
=====

SQLAlchemy包括一个事件API，该API发布了对SQLAlchemy核心和ORM内部的广泛挂钩。

事件注册
------------------

通过一个单一的API点注册事件，即   :func:`.listen`  函数，或者使用装饰器   :func:` .listens_for` 。这些函数接受目标参数，一个用于标识要拦截的事件字符串标识符，以及一个用户定义的监听函数。这两个函数的其他位置和关键字参数可能会被特定类型事件所支持，这些事件可能会为给定的事件函数指定备选接口，或者根据给定的目标提供有关次要事件目标的说明。

事件的名称和对应的监听者函数的参数签名是从一个绑定到标记类的规范方法中推导出来的，这个规范方法在文档中有详细的描述。例如，对于  :meth:`_events.PoolEvents.connect`  的文档说明了事件名称是 ` `"connect"``，并且用户定义的监听函数应该接收两个位置参数::

    from sqlalchemy.event import listen
    from sqlalchemy.pool import Pool


    def my_on_connect(dbapi_con, connection_record):
        print("新的DBAPI连接:", dbapi_con)


    listen(Pool, "connect", my_on_connect)

使用装饰器   :func:`.listens_for`  进行监听::

    from sqlalchemy.event import listens_for
    from sqlalchemy.pool import Pool


    @listens_for(Pool, "connect")
    def my_on_connect(dbapi_con, connection_record):
        print("新的DBAPI连接:", dbapi_con)

.. _event_named_argument_styles:

命名参数样式
---------------------

有一些参数样式的变体可以被监听函数接受。以  :meth:`_events.PoolEvents.connect`  为例，此函数文档中有关于接受 ` `dbapi_connection`` 和 ``connection_record`` 参数的描述。我们可以通过建立一个接受 ``**keyword`` 参数的监听者函数，通过在   :func:`.listen`  或   :func:` .listens_for`  中传递 ``named=True``，来按名称接收这些参数值。

    from sqlalchemy.event import listens_for
    from sqlalchemy.pool import Pool


    @listens_for(Pool, "connect", named=True)
    def my_on_connect(**kw):
        print("新的DBAPI连接:", kw["dbapi_connection"])

使用命名参数传递时，函数参数说明中列出的名称将用作字典中的键。

命名样式通过名称传递所有参数，而不管函数签名，因此可以列出特定的参数，并且可以以任何顺序列出，只要名称匹配即可::

    from sqlalchemy.event import listens_for
    from sqlalchemy.pool import Pool


    @listens_for(Pool, "connect", named=True)
    def my_on_connect(dbapi_connection, **kw):
        print("新的DBAPI连接:", dbapi_connection)
        print("连接记录:", kw["connection_record"])

上面，存在 ``**kw`` 告诉   :func:`.listens_for` ，参数应该通过名称而不是位置传递给函数。

目标
-------

  :func:`.listen`  函数对于目标非常灵活。 它通常接受类、这些类的实例以及相关的类或对象，从中可以派生出适当的目标。例如，上述提到的 ` `"connect"`` 事件接受   :class:`_engine.Engine`  类和对象，以及   :class:` _pool.Pool`  类和对象：

    from sqlalchemy.event import listen
    from sqlalchemy.pool import Pool, QueuePool
    from sqlalchemy import create_engine
    from sqlalchemy.engine import Engine
    import psycopg2


    def connect():
        return psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")


    my_pool = QueuePool(connect)
    my_engine = create_engine("postgresql+psycopg2://ed@localhost/test")

    # 将监听事件与所有的Pool实例关联
    listen(Pool, "connect", my_on_connect)

    # 通过Engine类将监听事件与所有的Pool实例关联
    listen(Engine, "connect", my_on_connect)

    # 将监听事件与my_pool关联
    listen(my_pool, "connect", my_on_connect)

    # 将监听事件与my_engine.pool关联
    listen(my_engine, "connect", my_on_connect)

.. _event_modifiers:

修改器
---------

一些监听器允许在   :func:`.listen`  中传递修改器。这些修饰符有时会为监听器提供备用的调用签名。例如，在ORM事件中，一些事件监听器可以有一个返回值，该返回值可以修改后续处理。默认情况下，没有任何监听器需要返回值，但是通过传递 ` `retval=True``，可以支持此值返回::

    def validate_phone(target, value, oldvalue, initiator):
        """Strip非数字字符从电话号码中"""

        return re.sub(r"\D", "", value)


    # 在UserContact.phone属性上设置监听器，以使用返回值
    listen(UserContact.phone, "set", validate_phone, retval=True)

事件参考
---------------

SQLAlchemy Core和SQLAlchemy ORM都具有各种事件挂钩:

* **Core事件** - 这些在   :ref:`core_event_toplevel`  中描述
  ，包括特定于连接池生命周期、SQL语句执行、事务生命周期和模式创建和拆卸的事件挂钩。

* **ORM事件** - 这些在   :ref:`orm_event_toplevel`  中描述，包括特定于类和属性检测、对象初始化钩子、属性更改后钩子、会话状态、刷新和提交钩子、映射器初始化、对象/结果填充和每个实例的持久性钩子的事件挂钩。

API参考
-------------

.. autofunction:: sqlalchemy.event.listen

.. autofunction:: sqlalchemy.event.listens_for

.. autofunction:: sqlalchemy.event.remove

.. autofunction:: sqlalchemy.event.contains