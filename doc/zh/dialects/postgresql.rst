.. _postgresql_toplevel:

PostgreSQL
==========

.. automodule:: sqlalchemy.dialects.postgresql.base

数组类型
-----------

PostgreSQL 方言支持数组，可以作为多维列类型以及数组文字：

* :class:`_postgresql.ARRAY` - ARRAY 数据类型

* :class:`_postgresql.array` - 数组文字

* :func:`_postgresql.array_agg` - ARRAY_AGG SQL 函数

* :class:`_postgresql.aggregate_order_by` - 用于 PG 的 ORDER BY 聚合函数语法的帮助者。

JSON 类型
----------

PostgreSQL 方言支持 JSON 和 JSONB 数据类型，包括 psycopg2 的本地支持和对所有 PostgreSQL 特殊运算符的支持：

* :class:`_postgresql.JSON`

* :class:`_postgresql.JSONB`

* :class:`_postgresql.JSONPATH`

HSTORE 类型
-----------

支持 PostgreSQL 的 HSTORE 类型以及 hstore 文字：

* :class:`_postgresql.HSTORE` - HSTORE 数据类型

* :class:`_postgresql.hstore` - hstore 文字

枚举类型
----------

PostgreSQL 拥有独立的 TYPE 结构用于实现枚举类型。这种方法引入了在 SQLAlchemy 方面的显著复杂性，即何时应该创建和删除此类型。该类型对象也是一个独立的可反映实体。应查阅下面的部分：

* :class:`_postgresql.ENUM` - BOOL 的 DDL 和类型支持。

* :meth:`.PGInspector.get_enums` - 检索当前 ENUM 类型的列表。

* :meth:`.postgresql.ENUM.create` , :meth:`.postgresql.ENUM.drop` - 枚举的单独 CREATE 和 DROP 命令。

数组中使用枚举
------------------

目前，后端 DBAPI 不直接支持 ENUM 和 ARRAY 的组合。在 SQLAlchemy 1.3.17 之前，需要特殊解决方法才能允许这种组合工作，如下所述。

.. versionchanged:: 1.3.17 现在不需要任何解决方法，即可由 SQLAlchemy 的实现直接处理 ENUM 和 ARRAY 的组合。

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

例如：

.. sourcecode:: python

    Table(
        "mydata",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("data", ArrayOfEnum(ENUM("a", "b", "c", name="myenum"))),
    )

此类型未包含在内置类型中，因为它与突然决定在新版本中直接支持 ARRAY 的 ENUM 的 DBAPI 不兼容。

使用 JSON / JSONB 和 ARRAY
------------------------------

与使用 ENUM 一样，在 SQLAlchemy 1.3.17 之前，对于 JSON / JSONB 的 ARRAY，需要呈现适当的 CAST，以免发生错误。当前的 psycopg2 驱动程序可以正确地调整结果集，而不需要任何特殊步骤。

.. versionchanged:: 1.3.17 现在不需要使用任何解决方法，即可由 SQLAlchemy 的实现直接处理 JSON/JSONB 和 ARRAY 的组合。

.. sourcecode:: python

    class CastingArray(ARRAY):
        def bind_expression(self, bindvalue):
            return sa.cast(bindvalue, self)

例如：

.. sourcecode:: python

    Table(
        "mydata",
        metadata,
        Column("id", Integer, primary_key=True),
        Column("data", CastingArray(JSONB)),
    )

区域和多区间类型
--------------------------

PostgreSQL 区间和多区间类型可用于 psycopg、pg8000 和 asyncpg 方言；psycopg2 方言仅支持区间类型。

正在传递到数据库的数据值可以作为字符串值传递，也可以使用:class:`_postgresql.Range` 数据对象。

E.g. 一个使用完全类型化的模型示例:class:`_postgresql.TSRANGE` 数据类型::

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

下面是向上面的 room_booking 表中插入行的示例::

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

从任何范围列中选择也将返回 :class:`_postgresql.Range` 对象，如下所示：

    from sqlalchemy import select

    with Session(engine) as session:
        for row in session.execute(select(RoomBooking.during)):
            print(row)

可用的区间数据类型如下：

* :class:`_postgresql.INT4RANGE`
* :class:`_postgresql.INT8RANGE`
* :class:`_postgresql.NUMRANGE`
* :class:`_postgresql.DATERANGE`
* :class:`_postgresql.TSRANGE`
* :class:`_postgresql.TSTZRANGE`

Multiranges
^^^^^^^^^^^

多区间在 PostgreSQL 14 及以上版本中受支持。SQLAlchemy 的多区间数据类型将处理 :class:`_postgresql.Range` 类型的列表。

仅对 psycopg、asyncpg 和 pg8000 方言支持多区间，对于使用 SQLAlchemy 默认 "postgresql" 方言的 psycopg2 方言，不支持多区间数据类型。

.. versionadded:: 2.0 添加对 MULTIRANGE 数据类型的支持。
   SQLAlchemy 表示多区间值为 :class:`_postgresql.Range` 对象的列表。

下面的示例说明了 :class:`_postgresql.TSMULTIRANGE` 数据类型的用法：

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

插入和选择记录的演示：

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

.. 注意：在上面的示例中，ORM 处理的 :class:`_postgresql.Range` 类型列表将无法自动检测到特定列表值的就地更改；要使用 ORM 更新列表值，请将新列表分配给属性，或使用:class:`.MutableList` 类型修饰符。请参见 :ref:`mutable_toplevel` 部分以获得背景信息。


可用的多区间数据类型如下：

* :class:`_postgresql.INT4MULTIRANGE`
* :class:`_postgresql.INT8MULTIRANGE`
* :class:`_postgresql.NUMMULTIRANGE`
* :class:`_postgresql.DATEMULTIRANGE`
* :class:`_postgresql.TSMULTIRANGE`
* :class:`_postgresql.TSTZMULTIRANGE`


网络数据类型
------------------

包括的网络数据类型是 :class:`_postgresql.INET`、:class:`_postgresql.CIDR` 和 :class:`_postgresql.MACADDR`。

对于 :class:`_postgresql.INET` 和 :class:`_postgresql.CIDR` 数据类型，可以根据是否传递 Python ``ipaddress`` 对象，包括 ``ipaddress.IPv4Network``、``ipaddress.IPv6Network``、``ipaddress.IPv4Address``、``ipaddress.IPv6Address``，来有条件地支持这些数据类型。这种支持当前是 **DBAPI 的默认行为，并且因 DBAPI 而异。SQLAlchemy 尚未实现其自己的网络地址转换逻辑。**

* :ref:`postgresql_psycopg` 和 :ref:`postgresql_asyncpg` 完全支持这些数据类型；默认情况下，从 ``ipaddress`` 系列返回对象。

* :ref:`postgresql_psycopg2` 方言仅发送和接收字符串。

* :ref:`postgresql_pg8000` 方言仅为 :class:`_postgresql.INET` 数据类型使用字符串，而为 :class:`_postgresql.CIDR` 类型使用字符串和``ipaddress.IPv4Address`` 和 ``ipaddress.IPv6Address`` 对象。


要 **将所有上述 DBAPI 规范化为仅返回字符串**，请使用 ``native_inet_types`` 参数传递一个值为``False``：

    e = create_engine(
        "postgresql+psycopg://scott:tiger@host/dbname", native_inet_types=False
    )

使用以上参数，``psycopg``、``asyncpg`` 和 ``pg8000`` 方言将禁用 DBAPI 对这些类型的自适应性，并只返回字符串，与较旧的``psycopg2`` 方言行为相匹配。

该参数也可以设置为 ``True``，在它引发 ``NotImplementedError`` 的情况下，为那些不支持或尚未完全支持将行转换为 Python ``ipaddress`` 数据类型的后端（当前是 psycopg2 和 pg8000）。

.. versionadded:: 2.0.18 -- 添加了 ``native_inet_types`` 参数。

PostgreSQL 数据类型
---------------------

与所有方言一样，所有已知可与 PostgreSQL 一起使用的大写类型都可以从方言的顶层导入，无论它们是来自 :mod:`sqlalchemy.types` 还是来自本地方言：

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

特定于 PostgreSQL，或具有 PostgreSQL 特定构造参数的类型如下：

.. note: 当使用 :noindex: 时，表示不在本地方言模块中重新定义类型，仅从 sqltypes 导入。这避免了 sphinx 构建中的警告。

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

PostgreSQL SQL 元素和函数
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

PostgreSQL 约束类型
---------------------------

SQLAlchemy 通过 :class:`ExcludeConstraint` 类支持 PostgreSQL EXCLUDE 约束：

.. autoclass:: ExcludeConstraint
   :members: __init__

例如：

.. sourcecode:: python

    from sqlalchemy.dialects.postgresql import ExcludeConstraint, TSRANGE


    class RoomBooking(Base):
        __tablename__ = "room_booking"

        room = Column(Integer(), primary_key=True)
        during = Column(TSRANGE())

        __table_args__ = (ExcludeConstraint(("room", "="), ("during", "&&")),)

PostgreSQL DML 构造
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