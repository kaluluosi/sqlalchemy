.. _postgresql_toplevel:

PostgreSQL
==========

.. automodule:: sqlalchemy.dialects.postgresql.base

数组类型
-----------

PostgreSQL方言支持数组类型，包括多维列类型和数组字面值：

-   :class:`_postgresql.ARRAY`  - ARRAY数据类型
-   :class:`_postgresql.array`  - 数组字面值
-   :func:`_postgresql.array_agg`  - ARRAY_AGG SQL 函数
-   :class:`_postgresql.aggregate_order_by`  - PG的ORDER BY汇总函数语法助手

JSON类型
----------

PostgreSQL方言支持JSON和JSONB数据类型，包括psycopg2的本地支持和所有PostgreSQL的特殊运算符支持：

-   :class:`_postgresql.JSON` 
-   :class:`_postgresql.JSONB` 
-   :class:`_postgresql.JSONPATH` 

HSTORE类型
-----------

PostgreSQL HSTORE类型和hstore字面值都被支持：

-   :class:`_postgresql.HSTORE`  - HSTORE数据类型
-   :class:`_postgresql.hstore`  - hstore字面值

枚举类型
----------

PostgreSQL有一个独立的可创建的实现枚举类型的结构。这种方法在SQLAlchemy方面会引入显着复杂性，关于何时应该创建和删除此类型在SQLAlchemy端上是独立反射实体。应参考以下部分：

-   :class:`_postgresql.ENUM`  - 用于ENUM的DDL和类型支持。
-  :meth:`.PGInspector.get_enums`  - 检索当前ENUM类型的列表。
-  :meth:`.postgresql.ENUM.create` ,  :meth:` .postgresql.ENUM.drop`  - ENUM的单个CREATE和DROP命令。

使用ARRAY和枚举
^^^^^^^^^^^^^^^^^^^^^^^

由于后端DBAPI目前不直接支持ENUM和ARRAY的组合，因此在SQLAlchemy 1.3.17之前，需要一种特殊的解决方法才能使这种组合起作用：

- 在SQLAlchemy 1.3.17之前，必须使用特殊的解决方法才能使此组合起作用。
- 在SQLAlchemy 1.3.17中ENUM和ARRAY的组合得到了直接处理。

.. sourcecode:: python

    from sqlalchemy import TypeDecorator
    from sqlalchemy.dialects.postgresql import ARRAY


    class ArrayOfEnum(TypeDecorator):
        impl = ARRAY

        def bind_expression(self, bindvalue):
            return sa.cast(bindvalue, self)

        def result_processor(self, dialect, coltype):
            super_rp = super(ArrayOfEnum, self).result_processor(dialect, coltype)

            def handle_raw_string(value):
                inner = re.match(r"^{(.*)}$", value).group(1)
                return inner.split(",") if inner else []

            def process(value):
                if value is None:
                    return None
                return super_rp(handle_raw_string(value))

            return process

例如::

    Table(
        "mydata",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("data", ArrayOfEnum(ENUM("a", "b", "c", name="myenum"))),
    )

此类型不作为内置类型包含，因为它与DBAPI不兼容，如果在新版本中突然决定直接支持ENUM的ARRAY，将不兼容。

使用JSON/JSONB和数组
^^^^^^^^^^^^^^^^^^^^^^^^^^

类似于使用ENUM，但是在SQLAlchemy 1.3.17之前，针对JSON/JSONB数组，我们需要渲染适当的CAST，可以正确地处理当前psycopg2驱动程序返回的结果集，而无需进行任何特殊步骤。

.. versionchanged:: 1.3.17 JSON / JSONB和ARRAY的组合现在通过SQLAlchemy的实现直接处理，无需任何解决方法。

.. sourcecode:: python

    class CastingArray(ARRAY):
        def bind_expression(self, bindvalue):
            return sa.cast(bindvalue, self)

例如::

    Table(
        "mydata",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("data", CastingArray(JSONB)),
    )

范围和多范围类型
--------------------------

PostgreSQL范围和多范围类型是psycopg、pg8000和asyncpg方言中支持的；psycopg2方言仅支持范围类型。

将传递给数据库的数据值可以作为字符串值传递，也可以使用 :class:`_postgresql.Range` 数据对象。

.. versionadded:: 2.0  添加了后端不可知的 :class:`_postgresql.Range` 对象，用于指示范围。

例如，使用完全键入的模型的示例   :class:`_postgresql.TSRANGE`  数据类型::

    from datetime import datetime

    from sqlalchemy.dialects.postgresql import Range
    from sqlalchemy.dialects.postgresql import TSRANGE
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class RoomBooking(Base):
        __tablename__ = "room_booking"

        id: Mapped[int] = mapped_column(primary_key=True)
        room: Mapped[str]
        during: Mapped[Range[datetime]] = mapped_column(TSRANGE)

如上所述， :class:`_postgresql.Range` 类型是一个简单的数据类，将表示范围的边界。以下示例说明了将一行添加到“room_booking”表中的情况：

    from sqlalchemy import create_engine
    from sqlalchemy.orm import Session

    engine = create_engine("postgresql+psycopg://scott:tiger@pg14/dbname")

    Base.metadata.create_all(engine)

    with Session(engine) as session:
        booking = RoomBooking(
            room="101", during=Range(datetime(2013, 3, 23), datetime(2013, 3, 25))
        )
        session.add(booking)
        session.commit()

从任何范围列中进行选择还将返回 :class:`_postgresql.Range` 对象。

可用的更范围数据类型如下：

*   :class:`_postgresql.INT4RANGE` 
*   :class:`_postgresql.INT8RANGE` 
*   :class:`_postgresql.NUMRANGE` 
*   :class:`_postgresql.DATERANGE` 
*   :class:`_postgresql.TSRANGE` 
*   :class:`_postgresql.TSTZRANGE` 

.. autoclass:: sqlalchemy.dialects.postgresql.Range
    :members:

多范围
^^^^^^^^^^^

PostgreSQL 14及以上版本支持多范围。SQLAlchemy的多范围数据类型处理 :class:`_postgresql.Range` 类型列表。

只有psycopg、asyncpg和pg8000方言支持多负载。psycopg2方言，即SQLAlchemy的默认“postgresql”方言，不支持多范围数据类型。

.. versionadded:: 2.0 添加了对 MULTIRANGE数据类型的支持。

以下示例说明了使用 :class:`_postgresql.TSMULTIRANGE` 数据类型的情况：

    from datetime import datetime
    from typing import List

    from sqlalchemy.dialects.postgresql import Range
    from sqlalchemy.dialects.postgresql import TSMULTIRANGE
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class EventCalendar(Base):
        __tablename__ = "event_calendar"

        id: Mapped[int] = mapped_column(primary_key=True)
        event_name: Mapped[str]
        in_session_periods: Mapped[List[Range[datetime]]] = mapped_column(TSMULTIRANGE)

说明插入和选择记录：

    from sqlalchemy import create_engine
    from sqlalchemy import select
    from sqlalchemy.orm import Session

    engine = create_engine("postgresql+psycopg://scott:tiger@pg14/test")

    Base.metadata.create_all(engine)

    with Session(engine) as session:
        calendar = EventCalendar(
            event_name="SQLAlchemy Tutorial Sessions",
            in_session_periods=[
                Range(datetime(2013, 3, 23), datetime(2013, 3, 25)),
                Range(datetime(2013, 4, 12), datetime(2013, 4, 15)),
                Range(datetime(2013, 5, 9), datetime(2013, 5, 12)),
            ],
        )
        session.add(calendar)
        session.commit()

        for multirange in session.scalars(select(EventCalendar.in_session_periods)):
            for range_ in multirange:
                print(f"Start: {range_.lower}  End: {range_.upper}")

..注：在上面的示例中，通过ORM处理的  :class:`_postgresql.Range` .MutableList` 类型修饰符。有关详细信息，请参见：ref：‘mutable_toplevel’。

可用的多范围数据类型如下所示：

*   :class:`_postgresql.INT4MULTIRANGE` 
*   :class:`_postgresql.INT8MULTIRANGE` 
*   :class:`_postgresql.NUMMULTIRANGE` 
*   :class:`_postgresql.DATEMULTIRANGE` 
*   :class:`_postgresql.TSMULTIRANGE` 
*   :class:`_postgresql.TSTZMULTIRANGE` 

.. _postgresql_network_datatypes:

网络数据类型
------------------

已包含的网络数据类型是  :class:`_postgresql.INET` 、  :class:` _postgresql.CIDR` 。

有条件支持将数据类型  :class:`_postgresql.INET` ` ipaddress``对象，包括``ipaddress.IPv4Network``、``ipaddress.IPv6Network``、``ipaddress.IPv4Address``和``ipaddress.IPv6Address``等对象。目前这种支持是**DBAPI本身的默认行为，并因DBAPI而异，SQLAlchemy尚未实现其自己的网络地址转换逻辑**。

-   :ref:`postgresql_psycopg`  和   :ref:` postgresql_asyncpg`  完全支持这些类型；默认情况下从行中返回``ipaddress``系列的对象。
- 方言   :ref:`postgresql_psycopg2`  只发送和检索字符串。
- 方言   :ref:`postgresql_pg8000`  支持   :class:` _postgresql.INET`  数据类型的 ``ipaddress.IPv4Address`` 和`ipaddress.IPv6Address`` 对象，但是在  :class:`_postgresql.CIDR` 类型中使用字符串。

要**将所有上述DBAPI规范化为仅返回字符串**，请使用 ``native_inet_types`` 参数，传递值``False``：

    e = create_engine(
        "postgresql+psycopg://scott:tiger@host/dbname", native_inet_types=False
    )

使用上述参数，``psycopg``、``asyncpg`` 和 ``pg8000`` 方言将禁用DBAPI对这些类型的适应，仅返回字符串，与更老的``psycopg2``方言的行为匹配。

该参数也可以设置为``True``，其中它的影响是对那些不支持或暂未完全支持将行转换为Python ``ipaddress``数据类型（当前为 psycopg2 和 pg8000）的后端引发``NotImplementedError``。

.. versionadded:: 2.0.18 添加了``native_inet_types``参数。

PostgreSQL 数据类型
---------------------

与所有SQLAlchemy方言一样，已知与PostgreSQL有效的所有大写类型都可以从方言的顶层导入，无论它们是来自:mod：`sqlalchemy.types` 还是来自本地方言::

    from sqlalchemy.dialects.postgresql import (
        ARRAY,
        BIGINT,
        BIT,
        BOOLEAN,
        BYTEA,
        CHAR,
        CIDR,
        CITEXT,
        DATE,
        DOUBLE_PRECISION,
        ENUM,
        FLOAT,
        HSTORE,
        INET,
        INTEGER,
        INTERVAL,
        JSON,
        JSONB,
        MACADDR,
        MACADDR8,
        MONEY,
        NUMERIC,
        OID,
        REAL,
        SMALLINT,
        TEXT,
        TIME,
        TIMESTAMP,
        UUID,
        VARCHAR,
        INT4RANGE,
        INT8RANGE,
        NUMRANGE,
        DATERANGE,
        TSRANGE,
        TSTZRANGE,
        REGCONFIG,
        REGCLASS,
        TSQUERY,
        TSVECTOR,
    )

特定于 PostgreSQL 的类型，或具有特定于 PostgreSQL 的构造参数的类型，如下所示：

.. note:使用 :noindex: 表示不在方言模块中重新定义的类型，只是从sqltypes导入。 这避免了在sphinx构建中出现警告。

.. currentmodule:: sqlalchemy.dialects.postgresql

.. autoclass:: sqlalchemy.dialects.postgresql.AbstractRange
    :members: comparator_factory

.. autoclass:: sqlalchemy.dialects.postgresql.AbstractMultiRange


.. autoclass:: ARRAY
    :members: __init__, Comparator


.. autoclass:: BIT

.. autoclass:: BYTEA
    :members: __init__

.. autoclass:: CIDR

.. autoclass:: CITEXT

.. autoclass:: DOMAIN
    :members: __init__, create, drop

.. autoclass:: DOUBLE_PRECISION
    :members: __init__
    :noindex:


.. autoclass:: ENUM
    :members: __init__, create, drop


.. autoclass:: HSTORE
    :members:


.. autoclass:: INET

.. autoclass:: INTERVAL
    :members: __init__

.. autoclass:: JSON
    :members:

.. autoclass:: JSONB
    :members:

.. autoclass:: JSONPATH

.. autoclass:: MACADDR

.. autoclass:: MACADDR8

.. autoclass:: MONEY

.. autoclass:: OID

.. autoclass:: REAL
    :members: __init__
    :noindex:


.. autoclass:: REGCONFIG

.. autoclass:: REGCLASS

.. autoclass:: TIMESTAMP
    :members: __init__

.. autoclass:: TIME
    :members: __init__

.. autoclass:: TSQUERY

.. autoclass:: TSVECTOR

.. autoclass:: UUID
    :members: __init__
    :noindex:


.. autoclass:: INT4RANGE


.. autoclass:: INT8RANGE


.. autoclass:: NUMRANGE


.. autoclass:: DATERANGE


.. autoclass:: TSRANGE


.. autoclass:: TSTZRANGE


.. autoclass:: INT4MULTIRANGE


.. autoclass:: INT8MULTIRANGE


.. autoclass:: NUMMULTIRANGE


.. autoclass:: DATEMULTIRANGE


.. autoclass:: TSMULTIRANGE


.. autoclass:: TSTZMULTIRANGE


PostgreSQL SQL元素和函数
--------------------------------------

.. autoclass:: aggregate_order_by

.. autoclass:: array

.. autofunction:: array_agg

.. autofunction:: Any

.. autofunction:: All

.. autoclass:: hstore
    :members:

.. autoclass:: to_tsvector

.. autoclass:: to_tsquery

.. autoclass:: plainto_tsquery

.. autoclass:: phraseto_tsquery

.. autoclass:: websearch_to_tsquery

.. autoclass:: ts_headline

PostgreSQL约束类型
---------------------------

SQLAlchemy通过 :class:`_postgresql.ExcludeConstraint` 类支持PostgreSQL EXCLUDE约束：

.. autoclass:: ExcludeConstraint
   :members: __init__

例如：

    from sqlalchemy.dialects.postgresql import ExcludeConstraint, TSRANGE


    class RoomBooking(Base):
        __tablename__ = "room_booking"

        room = Column(Integer(), primary_key=True)
        during = Column(TSRANGE())

        __table_args__ = (ExcludeConstraint(("room", "="), ("during", "&&")),)

PostgreSQL DML构造
-------------------------

.. autofunction:: sqlalchemy.dialects.postgresql.insert

.. autoclass:: sqlalchemy.dialects.postgresql.Insert
  :members:

.. _postgresql_psycopg2:

psycopg2
--------

.. automodule:: sqlalchemy.dialects.postgresql.psycopg2

.. _postgresql_psycopg:

psycopg
--------

.. automodule:: sqlalchemy.dialects.postgresql.psycopg

.. _postgresql_pg8000:

pg8000
------

.. automodule:: sqlalchemy.dialects.postgresql.pg8000

.. _dialect-postgresql-asyncpg:

.. _postgresql_asyncpg:

asyncpg
-------

.. automodule:: sqlalchemy.dialects.postgresql.asyncpg

psycopg2cffi
------------

.. automodule:: sqlalchemy.dialects.postgresql.psycopg2cffi