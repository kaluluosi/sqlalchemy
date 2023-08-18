.. currentmodule:: sqlalchemy.types

.. _types_custom:

自定义类型
============

有多种方法可重新定义现有类型的行为，以及提供新的类型。

覆盖类型编译
---------------------------

频繁需求是强制一个类型的“字符串”版本，也就是在CREATE TABLE语句或其他SQL函数（如CAST）中呈现的版本被更改。例如，应用程序可能希望强制在除一个外的所有平台上都采用“BINARY”来呈现，而在其中一个应用程序中则希望呈现“BLOB”。对于大多数用例，最好使用现有的通用类型（在本例中为：class:`~sqlalchemy.types.LargeBinary`），但要更准确地控制类型，可以将每方言都有一个编译指令与任何类型相关联：

    from sqlalchemy.ext.compiler import compiles
    from sqlalchemy.types import BINARY


    @compiles(BINARY, "sqlite")
    def compile_binary_sqlite(type_, compiler, **kw):
        return "BLOB"

上述代码允许使用:class:`~sqlalchemy.types.BINARY`，它将生成字符串'BINARY'（在所有后端除SQLite外），在此情况下它将生成“BLOB”。

有关其他示例，请参阅:ref:`type_compilation_extension` 部分，.:ref:`sqlalchemy.ext.compiler_toplevel` 下面的子部分。

.. _types_typedecorator:

增强现有类型
-------------------------

:class:`~sqlalchemy.types.TypeDecorator`允许创建自定义类型，它将添加绑定参数和结果处理行为到现有的类型对象中。当需要对数据进行额外的Python-to-db marshal（也就是从和向应用程序编排数据库）时使用它。

.. note::

  :class:`~sqlalchemy.types.TypeDecorator`的绑定和结果处理是**附加的**，已由托管类型执行处理 :class:`~sqlalchemy.types.TypeDecorator`。每个DBAPI都是特定bot DBAPI进行处理的。虽然可以通过直接子类化来替换给定类型的此处理，但在实践中不需要，SQLAlchemy 公开了现在不支持此类公共使用情况。

.. topic:: ORM 提示

   :class:`~sqlalchemy.types.TypeDecorator`可用于提供一种一致的方式，用于将某种类型的值转换为传递到和从数据库传出。使用ORM时，使用与ORM无关的技术可以用于转换用户数据，这是使用 :func:`~sqlalchemy.orm.validates`装饰器。当需要将通过ORM模型传入的数据规范化为特定于业务的某种方式时，此技术可能更合适，并且不如数据类型通用。


.. autoclass:: TypeDecorator
   :members:

   .. autoattribute:: cache_ok

TypeDecorator Recipes
---------------------

一些关键的:class:`~sqlalchemy.types.TypeDecorator`示例如下。

.. _coerce_to_unicode:

将编码的字符串转换为 Unicode
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

有关:class:`~sqlalchemy.Unicode`类型的常见一个问题是，它只处理Python中的``unicode``对象。Python 2使用时，表示为``u'some string'``的值（如果不是3）是传递给绑定参数必须的对象格式。它执行的编码/解码函数仅适合DBAPI使用，是一个私有实现细节。

如果需要一个类型，可以安全地接收Python字节串，即包含非ASCII字符并且不是Python 2中的“u'” 对象的字符串，则可以使用:class:`~sqlalchemy.types.TypeDecorator`进行需要的强制转换：

    from sqlalchemy.types import TypeDecorator, Unicode


    class CoerceUTF8(TypeDecorator):
        """Safely coerce Python bytestrings to Unicode
        before passing off to the database."""

        impl = Unicode

        def process_bind_param(self, value, dialect):
            if isinstance(value, str):
                value = value.decode("utf-8")
            return value

四舍五入数值型
^^^^^^^^^^^^^^^^^

某些数据库连接器（如SQL Server）会在传递太多小数位数的Decimal时出现问题。下面是一个按原样舍入它们的配方：

    from sqlalchemy.types import TypeDecorator, Numeric
    from decimal import Decimal


    class SafeNumeric(TypeDecorator):
        """Adds quantization to Numeric."""

        impl = Numeric

        def __init__(self, *arg, **kw):
            TypeDecorator.__init__(self, *arg, **kw)
            self.quantize_int = -self.impl.scale
            self.quantize = Decimal(10) ** self.quantize_int

        def process_bind_param(self, value, dialect):
            if isinstance(value, Decimal) and value.as_tuple()[2] < self.quantize_int:
                value = value.quantize(self.quantize)
            return value

以UTC时区不带时区的形式存储时区感知时间戳
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

时间戳始终应以时区无关的方式存储在数据库中。对于大多数数据库，这意味着在存储之前应该首先将时间戳设置在UTC时区中，并且存储在没有与之关联的时区-naive（也就是，没有与之关联的任何时区；假定UTC是“隐式”时区）中。另外，数据库特定的类型，如PostgreSQL的“TIMESTAMP WITH TIMEZONE”通常更受欢迎，因为它们具有更丰富的功能；但是，在不存在或不喜欢时区智能数据库类型时，:class:`~sqlalchemy.types.TypeDecorator`可以用于创建将时区感知时间戳转换为时区感知和反之的数据类型。在下面的示例 中，使用Python默认的``datetime.timezone.utc``时区来规范化和取消规范化：

    import datetime


    class TZDateTime(TypeDecorator):
        impl = DateTime
        cache_ok = True

        def process_bind_param(self, value, dialect):
            if value is not None:
                if not value.tzinfo:
                    raise TypeError("tzinfo is required")
                value = value.astimezone(datetime.timezone.utc).replace(tzinfo=None)
            return value

        def process_result_value(self, value, dialect):
            if value is not None:
                value = value.replace(tzinfo=datetime.timezone.utc)
            return value

.. _custom_guid_type:

后端不受限制的 GUID 类型
^^^^^^^^^^^^^^^^^^^^^^^^^^

接收和返回Python uuid()对象。在使用PostgreSQL时，使用PG UUID类型，其他后端上使用 CHAR(32)，以十六进制字符串格式存储它们。如果需要，可以更改为将二进制存储在CHAR(16)中。

    from sqlalchemy.types import TypeDecorator, CHAR
    from sqlalchemy.dialects.postgresql import UUID
    import uuid


    class GUID(TypeDecorator):
        """Platform-independent GUID type.

        Uses PostgreSQL's UUID type, otherwise uses
        CHAR(32), storing as stringified hex values.

        """

        impl = CHAR
        cache_ok = True

        def load_dialect_impl(self, dialect):
            if dialect.name == "postgresql":
                return dialect.type_descriptor(UUID())
            else:
                return dialect.type_descriptor(CHAR(32))

        def process_bind_param(self, value, dialect):
            if value is None:
                return value
            elif dialect.name == "postgresql":
                return str(value)
            else:
                if not isinstance(value, uuid.UUID):
                    return "%.32x" % uuid.UUID(value).int
                else:
                    # hexstring
                    return "%.32x" % value.int

        def process_result_value(self, value, dialect):
            if value is None:
                return value
            else:
                if not isinstance(value, uuid.UUID):
                    value = uuid.UUID(value)
                return value

将 Python uuid.UUID 链接到自定义类型以进行 ORM 映射
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

使用 :ref:`Annotated Declarative Table <orm_declarative_mapped_column>`
映射声明ORM映射时，以上自定义“GUID”类型可以通过将其添加到
:class:`_orm.DeclarativeBase`类上通常定义的 :ref:`type annotation map <orm_declarative_mapped_column_type_map>` 来与Python`uuid.UUID`数据类型相关联：

    import uuid
    from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


    class Base(DeclarativeBase):
        type_annotation_map = {
            uuid.UUID: GUID,
        }

使用上述配置，扩展自“Base”的ORM映射类可以引用自动化地
使用``GUID``：

    class MyModel(Base):
        __tablename__ = "my_table"

        id: Mapped[uuid.UUID] = mapped_column(primary_key=True)

.. seealso::

    :ref:`orm_declarative_mapped_column_type_map`

采用 JSON 字符串
^^^^^^^^^^^^^^^^^^^^

此类型使用 ``simplejson`` 将Python数据结构编组到JSON。可以修改为使用Python内置的json编码器：

    from sqlalchemy.types import TypeDecorator, VARCHAR
    import json


    class JSONEncodedDict(TypeDecorator):
        """Represents an immutable structure as a json-encoded string.

        Usage:

            JSONEncodedDict(255)

        """

        impl = VARCHAR

        cache_ok = True

        def process_bind_param(self, value, dialect):
            if value is not None:
                value = json.dumps(value)

            return value

        def process_result_value(self, value, dialect):
            if value is not None:
                value = json.loads(value)
            return value

添加可变性
~~~~~~~~~~~~~~~~~

ORM默认不会检测此类型的“可变性” - 意思是，对值的原地更改将不会
检测到并不会被刷新。否则，您需要替换每个父对象上的现有价值以检测更改：：

    obj.json_value["key"] = "value"  # 在ORM中将 *不* 检测到

    obj.json_value = {"key": "value"}  # 在ORM中将 *检测* 到

如果父站上需要这种要求，则上述限制可能是正确的，因为许多
应用可能不需要一旦创建了值，就需要将其改变。 对于具有此要求的那些应用程序
最好使用``sqlalchemy.ext.mutable``扩展来支持可变性。
对于基于字典的JSON结构，我们可以使用此方法： 

    json_type = MutableDict.as_mutable(JSONEncodedDict)


    class MyClass(Base):
        #  ...

        json_data = Column(json_type)

.. seealso::

    :ref:`mutable_toplevel`

处理比较操作
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`~sqlalchemy.types.TypeDecorator`的默认行为是将表达式的“右侧”强制转换为相同的类型。对于像JSON这样的类型，这意味着任何操作符使用必须在JSON的条件下有意义。  对于一些情况，用户可能希望类型在某些情况下表现得像JSON，在其他情况下则表现得像纯文本。例如，如果希望处理JSON类型的LIKE操作符。 为处理像 ``JSONEncodedDict` ``类型这样的类型，我们需要在尝试使用此运算符之前使用 CAST 或 type_coerce来将列转换为文本形式：

    from sqlalchemy import type_coerce, String

    stmt = select(my_table).where(type_coerce(my_table.c.json_data, String).like("%foo%"))

:class:`~sqlalchemy.types.TypeDecorator`提供了一种内置系统，用于基于运算符构建类型转换，该系统基于运算符添加到 :class:`_expression.ColumnElement` 表达式构造中。：meth:`.TypeEngine.Comparator`类提供抽象比较对象以提供基本行为模板，许多类型提供自己的子实现。可以直接将用户定义的 :class:`.TypeEngine.Comparator` 实现构建到简单的类型子类中，以覆盖或定义新操作。下面，我们创建一个``Geometry``类型，它将应用PostGIS函数 ``ST_GeomFromText``到所有外向值，以及函数``ST_AsText``到所有传入数据： 

    from sqlalchemy import func
    from sqlalchemy.types import UserDefinedType


    class Geometry(UserDefinedType):
        def get_col_spec(self):
            return "GEOMETRY"

        def bind_expression(self, bindvalue):
            return func.ST_GeomFromText(bindvalue, type_=self)

        def column_expression(self, col):
            return func.ST_AsText(col, type_=self)

我们可以将 ``Geometry`` 类型应用于 :class:`_schema.Table` metadata 中，并在使用 :func:`_expression.select` 构造中使用它： 

    geometry = Table(
        "geometry",
        metadata,
        Column("geom_id", Integer, primary_key=True),
        Column("geom_data", Geometry),
    )

    print(
        select(geometry).where(
            geometry.c.geom_data == "LINESTRING(189412 252431,189631 259122)"
        )
    )

SQL语句将嵌入这两个函数，如适当。 ``ST_AsText``应用于列子句中，以便传入一个结果集之前将返回值运行通过该函数，而``ST_GeomFromText``应用于绑定参数，以便将传递值转换为： 

.. sourcecode:: sql

    SELECT geometry.geom_id, ST_AsText(geometry.geom_data) AS geom_data_1
    FROM geometry
    WHERE geometry.geom_data = ST_GeomFromText(:geom_data_2)

:meth:`.TypeEngine.column_expression` 方法与编译器的内部机制进行交互，以便SQL表达式不干扰包装表达式的标签化。因此，如果以表达式标签渲染 :func:`_expression.select` ，则将字符串标签移至包装表达式的外部： 

    print(select(geometry.c.geom_data.label("my_data")))

输出：

.. sourcecode:: sql

    SELECT ST_AsText(geometry.geom_data) AS my_data
    FROM geometry

另一个示例是我们装饰 :class:`_postgresql.BYTEA` 以提供``PGPString``，这将自动透明地使用PostgreSQL ``pgcrypto``扩展来加密/解密值：

    from sqlalchemy import (
        create_engine,
        String,
        select,
        func,
        MetaData,
        Table,
        Column,
        type_coerce,
        TypeDecorator,
    )

    from sqlalchemy.dialects.postgresql import BYTEA


    class PGPString(TypeDecorator):
        impl = BYTEA

        cache_ok = True

        def __init__(self, passphrase):
            super(PGPString, self).__init__()

            self.passphrase = passphrase

        def bind_expression(self, bindvalue):
            # convert the bind's type from PGPString to
            # String, so that it's passed to psycopg2 as is without
            # a dbapi.Binary wrapper
            bindvalue = type_coerce(bindvalue, String)
            return func.pgp_sym_encrypt(bindvalue, self.passphrase)

        def column_expression(self, col):
            return func.pgp_sym_decrypt(col, self.passphrase)


    metadata_obj = MetaData()
    message = Table(
        "message",
        metadata_obj,
        Column("username", String(50)),
        Column("message", PGPString("this is my passphrase")),
    )

    engine = create_engine("postgresql+psycopg2://scott:tiger@localhost/test", echo=True)
    with engine.begin() as conn:
        metadata_obj.create_all(conn)

        conn.execute(message.insert(), username="some user", message="this is my message")

        print(
            conn.scalar(select(message.c.message).where(message.c.username == "some user"))
        )

使用插入、选择语句和 :class:`_postgresql.BYTEA` 一起阅读“PGPString”类型时，将应用“pgp_sym_encrypt”和“pgp_sym_decrypt”函数：

.. sourcecode:: sql

    INSERT INTO message (username, message)
    VALUES (%(username)s, pgp_sym_encrypt(%(message)s, %(pgp_sym_encrypt_1)s))
    -- {'username': 'some user', 'message': 'this is my message',
    --  'pgp_sym_encrypt_1': 'this is my passphrase'}

    SELECT pgp_sym_decrypt(message.message, %(pgp_sym_decrypt_1)s) AS message_1
    FROM message
    WHERE message.username = %(username_1)s
    -- {'pgp_sym_decrypt_1': 'this is my passphrase', 'username_1': 'some user'}

.. _types_sql_value_processing:

应用 SQL 级别的绑定/结果处理
-----------------------------------------

如上所述， SQL 可用Python函数在参数发送到语句时以及从数据库加载结果行时进行调用。还可以在SQL级别定义转换。原因是仅当关系数据库包含特定系列的函数时，这些函数可能在应用程序和持久性格式之间的数据之间强制作用。例如，使用数据库定义的加密/解密函数，以及处理地理数据的存储过程。

任何 :class:`.TypeEngine`、:class:`.UserDefinedType` 或 :class:`.TypeDecorator` 子类都可以包含 :meth:`.TypeEngine.bind_expression` 和/或 :meth:`.TypeEngine.column_expression` 的实现。如果定义为返回一个非- ``None`` 的值，则应返回一个 :class:`_expression.ColumnElement` 表达式以注入到SQL语句中，无论是周围绑定参数还是列表达式都是如此。例如，为构建一个“几何”类型，将在向去除所有传出值应用PostGIS函数“ST_GeomFromText”和函数“ST_AsText”时，我们可以创建自己的:class:`~sqlalchemy.types.UserDefinedType`子类，该子类提供这些方法和功能`~sqlalchemy.sql.expression.func`： 

    from sqlalchemy import func
    from sqlalchemy.types import UserDefinedType


    class Geometry(UserDefinedType):
        def get_col_spec(self):
            return "GEOMETRY"

        def bind_expression(self, bindvalue):
            return func.ST_GeomFromText(bindvalue, type_=self)

        def column_expression(self, col):
            return func.ST_AsText(col, type_=self)

我们可以将“Geometry”类型应用于 :class:`_schema.Table` metadata ，并在 :func:`_expression.select` 中使用它： 

    geometry = Table(
        "geometry",
        metadata,
        Column("geom_id", Integer, primary_key=True),
        Column("geom_data", Geometry),
    )

    print(
        select(geometry).where(
            geometry.c.geom_data == "LINESTRING(189412 252431,189631 259122)"
        )
    )

结果SQL将嵌入两个函数。 根据情况，``ST_AsText``适用于列子句，以便将传入一个结果集之前将返回值运行通过该函数，而``ST_GeomFromText``应用于绑定参数，以便将传递的值转换为： 

.. sourcecode:: sql

    SELECT geometry.geom_id, ST_AsText(geometry.geom_data) AS geom_data_1
    FROM geometry
    WHERE geometry.geom_data = ST_GeomFromText(:geom_data_2)

:meth:`.TypeEngine.column_expression` 方法与编译器的内部机制进行交互，以便SQL表达式不干扰包装表达式的标签化。因此，如果以表达式标签渲染 :func:`_expression.select` ，则将字符串标签移至包装表达式的外部： 

    print(select(geometry.c.geom_data.label("my_data")))

输出：

.. sourcecode:: sql

    SELECT ST_AsText(geometry.geom_data) AS my_data
    FROM geometry

另一个示例是我们在 ：class:`.BYTEA` 内添加实现 PostgreSQL 阶乘运算符的功能。我们使用 :class:`.UnaryExpression` 构造与 :class:`_.custom_op` 的组合来生成阶乘表达式： 

    from sqlalchemy import Integer
    from sqlalchemy.sql.expression import UnaryExpression
    from sqlalchemy.sql import operators


    class MyInteger(Integer):
        class comparator_factory(Integer.Comparator):
            def factorial(self):
                return UnaryExpression(
                    self.expr, modifier=operators.custom_op("!"), type_=MyInteger
                )

使用上述类型： 

.. sourcecode:: sql

    >>> from sqlalchemy.sql import column
    >>> print(column("x", MyInteger).factorial())
    {printsql}x !

.. seealso::

    :meth:`.Operators.op`

    :attr:`.TypeEngine.comparator_factory`

创建新类型
------------------

:class:`~sqlalchemy.types.UserDefinedType`提供为定义全新的数据库类型的简单基类。使用此来表示SQLAlchemy不知道的原生数据库类型。如果只需要Python转换行为，请改用 :class:`~sqlalchemy.types.TypeDecorator`。

.. autoclass:: UserDefinedType
   :members:

   .. autoattribute:: cache_ok

.. _custom_and_decorated_types_reflection:

使用自定义类型和反射操作
-----------------------------------------

重要的是要注意，具有附加Python行为的数据库类型（包括基于:class:`.TypeDecorator`的类型以及其他数据类型的用户定义子类）在数据库架构内没有任何表示。使用数据库时元数据 introspection 功能，如在 :ref:`metadata_reflection` 中描述的，使用了将数据库服务器报告的数据类型信息与SQLAlchemy数据类型对象相关联的修补映射。例如，如果我们在PostgreSQL模式中查看特定数据库列的定义，我们可能会得到字符串“VARCHAR”。SQLAlchemy的 PostgreSQL方言有一个硬编码映射，将字符串名“VARCHAR”链接到SQLAlchemy的 :class:`~sqlalchemy.types.VARCHAR` 类，并且这就是为什么当我们发出像 ``Table('my_table', m, autoload_with=engine)` ` 这样的语句时，:class:`~sqlalchemy.schema.Column` 对象在其中将具有 :class:`~sqlalchemy.types.VARCHAR` 的实例。

这意味着，如果 :class:`~sqlalchemy.schema.Table` 对象使用类型对象，这些类型对象不直接对应于数据库本机类型名称，如果我们使用反射在其他地方依靠新的 :class:`_schema.MetaData` 集合创建新的 :class:`_schema.Table` 对象，则它将不会具有此数据类型。例如：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import (
    ...     Table,
    ...     Column,
    ...     MetaData,
    ...     create_engine,
    ...     PickleType,
    ...     Integer,
    ... )
    >>> metadata = MetaData()
    >>> my_table = Table(
    ...     "my_table", metadata, Column("id", Integer), Column("data", PickleType)
    ... )
    >>> engine = create_engine("sqlite://", echo="debug")
    >>> my_table.create(engine)
    {execsql}INFO sqlalchemy.engine.base.Engine
    CREATE TABLE my_table (
        id INTEGER,
        data BLOB
    )

以上，我们使用了 :class:`.PickleType`，它是一个 work on top of the :class:`.LargeBinary` datatype 的 :class:`.TypeDecorator` ，在SQLite是指数据库类型 ``BLOB``。在CREATE TABLE中，我们看到使用了“BLOB”数据类型。SQLite数据库不知道我们所使用的 :class:`.PickleType`。

如果我们查看 `my_table.c.data.type` 的数据类型，因为这是我们直接创建的Python对象，它是 :class:`.PickleType` ：

    >>> my_table.c.data.type
    PickleType()

但是，如果使用反射创建另一个 :class:`_schema.Table` 实例，则使用 :class:`.PickleType` 不会表示在我们创建的SQLite数据库中；我们得到 :class:`.BLOB` ：

.. sourcecode:: pycon+sql

    >>> metadata_two = MetaData()
    >>> my_reflected_table = Table("my_table", metadata_two, autoload_with=engine)
    {execsql}INFO sqlalchemy.engine.base.Engine PRAGMA main.table_info("my_table")
    INFO sqlalchemy.engine.base.Engine ()
    DEBUG sqlalchemy.engine.base.Engine Col ('cid', 'name', 'type', 'notnull', 'dflt_value', 'pk')通常，当应用程序使用自定义类型定义显式的`_schema.Table`元数据时，不需要使用表反射，因为必要的`_schema.Table`元数据已经存在。 然而，对于应用程序或它们的组合需要使用既包括自定义的Python级别数据类型，又设置其`_schema.Column`对象为从数据库反射而来的`_schema.Table`对象，仍需要表现出额外的Python行为，必须采取其他步骤。

其中最简单的方法是，如:ref:`reflection_overriding_columns`中所述，覆盖特定列。 通过这种技术，我们简单地使用反射以及针对我们要使用自定义或装饰数据类型的那些列的显式`_schema.Column`对象::

    >>> metadata_three = MetaData()
    >>> my_reflected_table = Table(
    ...     "my_table",
    ...     metadata_three,
    ...     Column("data", PickleType),
    ...     autoload_with=engine,
    ... )

上面的“my_reflected_table”对象是反射的，将从SQLite数据库加载“id”列的定义。 但是对于“data”列，我们已使用显式`_schema.Column`定义覆盖了反射对象，该定义包括我们想要的Python数据类型:class:`.PickleType`。 反射过程将保留这个`_schema.Column`对象::

    >>> my_reflected_table.c.data.type
    PickleType()

将数据库本地类型对象转换为自定义数据类型的一种更复杂的方法是使用`._DDLEvents.column_reflect`事件处理程序。 例如，如果我们知道所有`_schema.BLOB`数据类型实际上都要作为`_schema.PickleType`，则可以在全局范围内设置一条规则::

    from sqlalchemy import BLOB
    from sqlalchemy import event
    from sqlalchemy import PickleType
    from sqlalchemy import Table


    @event.listens_for(Table, "column_reflect")
    def _setup_pickletype(inspector, table, column_info):
        if isinstance(column_info["type"], BLOB):
            column_info["type"] = PickleType()

当上面的代码在任何表反射发生之前调用时（还要注意它应该在应用程序中**仅调用一次**，因为它是全局规则），仅反射包含`_schema.BLOB`数据类型的任何`_schema.Table`时，将将生成的数据类型存储在`_schema.Column`对象中作为`_schema.PickleType`。

在实践中，上述基于事件的方法可能需要其他规则，才能仅影响那些数据类型很重要的列，例如表名和可能的列名的查找表或其他启发式规则，以准确确定应使用哪些列一个Python数据类型。