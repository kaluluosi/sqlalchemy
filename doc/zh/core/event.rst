.. _event_toplevel:

事件
===

SQLAlchemy自带一个事件API，发布了各种钩子，包括对SQLAlchemy核心和ORM内部的钩子。

事件注册
--------

订阅事件是通过一个API点完成的：：func：` .listen`函数或者alternatively的：func：` .listens_for`。这些函数接受目标，一个标识符，用于标识要拦截的事件，和一个用户定义的监听函数。这两个函数的附加位置和关键字参数可以由特定类型的事件支持，这些事件可以指定给定事件函数的替代接口，或者根据给定目标提供关于二级事件目标的说明。

事件的名称和相应的监听函数的参数签名源自一个绑定到标记类的规范化方法，该标记类在文档中描述。例如, :meth:`_events.PoolEvents.connect`的文档表明事件名称为``"connect"``,并且用户定义的监听函数应该接收两个位置参数::

    from sqlalchemy.event import listen
    from sqlalchemy.pool import Pool


    def my_on_connect(dbapi_con, connection_record):
        print("New DBAPI connection:", dbapi_con)


    listen(Pool, "connect", my_on_connect)

使用:func: `.listens_for`装饰器进行监听,如下所示::

    from sqlalchemy.event import listens_for
    from sqlalchemy.pool import Pool


    @listens_for(Pool, "connect")
    def my_on_connect(dbapi_con, connection_record):
        print("New DBAPI connection:", dbapi_con)

.. _event_named_argument_styles:

命名参数风格
---------------------

有一些参数风格的变化可以被监听函数所接受。
以:meth:`_events.PoolEvents.connect`为例，此函数
记录为接收“dbapi_connection”和“connection_record”参数.
我们可以选择通过名称接收这些参数，建立一个监听器函数
接受``**keyword``参数，通过传递``named=True``来实现
:func:`.listen` 或:func:`.listens_for`::

    from sqlalchemy.event import listens_for
    from sqlalchemy.pool import Pool


    @listens_for(Pool, "connect", named=True)
    def my_on_connect(**kw):
        print("New DBAPI connection:", kw["dbapi_connection"])

使用命名参数传递时，参数规范中列出的名称将用作字典中的键。

命名风格将所有参数按名称传递，而不考虑函数签名，因此也可以列出特定参数，以任何顺序，只要名称匹配即可::

    from sqlalchemy.event import listens_for
    from sqlalchemy.pool import Pool


    @listens_for(Pool, "connect", named=True)
    def my_on_connect(dbapi_connection, **kw):
        print("New DBAPI connection:", dbapi_connection)
        print("Connection record:", kw["connection_record"])

上述代码中，“**kw”的存在告诉:func:`.listens_for`，应该通过名称将参数传递给函数，而不是按顺序传递。

目标
------

:func: `.listen` 函数针对目标非常灵活。它
通常接受类，这些类的实例和相关的
类或对象，可以从中推导出适当的目标。
例如，上面提到的“connect”事件接受
中的:class:`_engine.Engine`类和对象，以及:class:`_pool.Pool`类和对象

    from sqlalchemy.event import listen
    from sqlalchemy.pool import Pool, QueuePool
    from sqlalchemy import create_engine
    from sqlalchemy.engine import Engine
    import psycopg2


    def connect():
        return psycopg2.connect(user="ed", host="127.0.0.1", dbname="test")


    my_pool = QueuePool(connect)
    my_engine = create_engine("postgresql+psycopg2://ed@localhost/test")

    # associate listener with all instances of Pool
    listen(Pool, "connect", my_on_connect)

    # associate listener with all instances of Pool
    # via the Engine class
    listen(Engine, "connect", my_on_connect)

    # associate listener with my_pool
    listen(my_pool, "connect", my_on_connect)

    # associate listener with my_engine.pool
    listen(my_engine, "connect", my_on_connect)

.. _event_modifiers:

修饰符
---------

一些监听器允许向:func:`.listen`传递修饰符。
这些修饰符有时提供了替代调用签名的方法
侦听器。就像ORM事件一样，一些事件监听器可以有一个
返回值来修改后续处理。默认情况下，没有
听众需要返回值，但通过传递``retval=True``
可以支持这个值::

    def validate_phone(target, value, oldvalue, initiator):
        """Strip non-numeric characters from a phone number"""

        return re.sub(r"\D", "", value)


    # setup listener on UserContact.phone attribute, instructing
    # it to use the return value
    listen(UserContact.phone, "set", validate_phone, retval=True)

事件参考
---------------

SQLAlchemy Core和SQLAlchemy ORM都具有各种事件钩子:

* **核心事件** - 这些在中描述
  :ref:`core_event_toplevel`并包括特定于的事件钩子
  连接池生命周期，SQL语句执行，
  事务生命周期和模式创建和拆除。

* **ORM事件** - 这些在中描述
  :ref:`orm_event_toplevel` 并包括特定于的事件钩子
  类和属性插装，对象初始化
  钩子， 属性更改钩子，会话状态，流，以及
  提交挂钩，映射器初始化，对象/结果填充，
  以及每个实例的持久性钩子。

API
-------------

.. autofunction:: sqlalchemy.event.listen

.. autofunction:: sqlalchemy.event.listens_for

.. autofunction:: sqlalchemy.event.remove

.. autofunction:: sqlalchemy.event.contains