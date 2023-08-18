.. currentmodule:: sqlalchemy.types

.. _types_custom:

自定义类型
============

现有类型的行为有多种方法可以重新定义，同时也有办法提供新的类型。

覆盖类型编译
---------------------------

经常需要强制类型的“字符串”版本，即在CREATE TABLE语句或其他SQL函数(如CAST)中呈现的版本改变。例如，应用程序可能想要强制对所有平台(除了一种以外)渲染二进制，其中需要渲染BLOB。对于大多数用例，建议使用现有的通用类型，例如  :class:`.LargeBinary` 。但是，为了更精确地控制类型，可以将针对每个dialect的编译指令与任何类型相关联::

    from sqlalchemy.ext.compiler import compiles
    from sqlalchemy.types import BINARY


    @compiles(BINARY, "sqlite")
    def compile_binary_sqlite(type_, compiler, **kw):
        return "BLOB"

上面的代码允许使用  :class:`_types.BINARY` ，该类型将在除SQLite外的所有后端中产生字符串"BINARY"，在SQLite的情况下，它将生成字符串"BLOB"。

查看 :ref:`类型编译扩展` 关于更多例子的细节。

.. _types_typedecorator:

增强现有类型
-------------------------

：class:`.TypeDecorator`允许创建自定义类型，为现有类型添加绑定参数和结果处理行为。当需要在Python中进行额外的  :term:`marshalling`  时，就会使用它，以及从数据库中。

.. note::

   :class:`.TypeDecorator` 的bind和结果处理是**附加**于托管的类型的处理，这个处理是根据DBAPI基础上来的。虽然可以通过直接子类化替换给定类型的处理，但实际上从不需要以这种方式，SQLAlchemy不再支持这种公共用例。

.. topic:: ORM提示

     :class:`.TypeDecorator` .validates` 装饰器。这种技术可能更适用于当业务情况需要规范化数据时，而不是与数据类型一样通用。

.. autoclass:: TypeDecorator
   :members:

   .. autoattribute:: cache_ok

TypeDecorator配方
---------------------

以下是一些关键的 :class:`.TypeDecorator` 配方。

.. _coerce_to_unicode:

强制编码字符串为Unicode
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`.Unicode` ` unicode``对象，这意味着在Python2中，如果使用Python的字符串对象，那么将抛出异常。它所执行的编码/解码函数只是为了适应所使用的DBAPI所需要的格式，是主要的私有实现细节。

这可以通过使用 :class:`.TypeDecorator` 来实现，需要时进行必要的强制类型转换::

    from sqlalchemy.types import TypeDecorator, Unicode


    class CoerceUTF8(TypeDecorator):
        """将Python的bytestring的安全强制转换为Unicode，然后转发到数据库中。"""

        impl = Unicode

        def process_bind_param(self, value, dialect):
            if isinstance(value, str):
                value = value.decode("utf-8")
            return value

保留数字
^^^^^^^^^^^^^^^^^

某些数据库连接器（例如SQL Server）如果传递带有太多小数位的Decimal，则会报错。这里有一个将Decimal四舍五入的配方::

    from sqlalchemy.types import TypeDecorator, Numeric
    from decimal import Decimal


    class SafeNumeric(TypeDecorator):
        """增加了分量的数字。"""

        impl = Numeric

        def __init__(self, *arg, **kw):
            TypeDecorator.__init__(self, *arg, **kw)
            self.quantize_int = -self.impl.scale
            self.quantize = Decimal(10) ** self.quantize_int

        def process_bind_param(self, value, dialect):
            if isinstance(value, Decimal) and value.as_tuple()[2] < self.quantize_int:
                value = value.quantize(self.quantize)
            return value

存储时区感知时间戳为时区感知的UTC
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Relational数据库中的时间戳应始终以与时区无关的方式存储。对于大多数数据库，这意味着需要先将时间戳设置为UTC时区，然后存储它的时区为无时区（即，没有任何时区与之相关），UTC被假定为“隐式”时区。另外，也有特定于数据库的类型，如PostgreSQL的“TIMESTAMP WITH TIMEZONE”，通常是首选的，因为它们具有更丰富的功能。如果不能使用时间智能数据库类型，或者不喜欢该项功能，那么可以使用  :class:`.TypeDecorator` ` datetime.timezone.utc``时区被用于归一化和反归一化::

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

跨数据库的GUID类型
^^^^^^^^^^^^^^^^^^^^^^^^^^

接收并返回Python uuid()对象。在使用PostgreSQL的情况下使用PG UUID类型，在其他后端使用CHAR(32)，将它们以字符串形式存储为十六进制。如果需要，可以修改为将二进制存储在CHAR(16)中::

    from sqlalchemy.types import TypeDecorator, CHAR
    from sqlalchemy.dialects.postgresql import UUID
    import uuid


    class GUID(TypeDecorator):
        """与平台无关的GUID类型。

        在PostgreSQL中使用UUID类型，否则使用CHAR(32)，
        将字符串化的十六进制值存储。

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

链接Python ``uuid.UUID``到ORM映射自定义类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在使用  :ref:`装饰式ORM <orm_declarative_mapped_column>` 
映射时，上面定义的自定义``GUID``类型可能会与
Python ``uuid.UUID``数据类型相关联，并将其自动用于
的资源将在ORM映射到的列相关联该对象，则可以将其添加到
  :type:`_orm.DeclarativeBase`  类上通常定义的  :ref:` 类型注释映射 <orm_declarative_mapped_column_type_map>` ：

    import uuid
    from sqlalchemy.orm import DeclarativeBase, Mapped, mapped_column


    class Base(DeclarativeBase):
        type_annotation_map = {
            uuid.UUID: GUID,
        }

使用以上配置，扩展自``Base``的ORM映射类可以引用Python ``uuid.UUID``。自动使用``GUID``：

    class MyModel(Base):
        __tablename__ = "my_table"

        id: Mapped[uuid.UUID] = mapped_column(primary_key=True)

.. seealso::

      :ref:`orm_declarative_mapped_column_type_map` 

Marshal JSON Strings
^^^^^^^^^^^^^^^^^^^^

此类型使用“simplejson”将Python数据结构编组为/从JSON中。可修改为使用Python的内置json编码器::

    from sqlalchemy.types import TypeDecorator, VARCHAR
    import json


    class JSONEncodedDict(TypeDecorator):
        """将不可变结构表示为json编码string。用法：JSONEncodedDict(255)"""

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

ORM默认情况下不会检测这种类型的任何“可变性”，也就是说，不会检测任何值的就地更改并使其刷新。如果没有进一步的步骤，您需要替换每个父对象上的现有值以检测更改::  obj.json_value["key"] = "value"  # ORM不会检测  obj.json_value = {"key": "value"}  # ORM会检测 

上面的限制最好使用``sqlalchemy.ext.mutable``扩展实现可变性。 对于以字典为导向的JSON结构，我们可以将其应用为：

    json_type = MutableDict.as_mutable(JSONEncodedDict)


    class MyClass(Base):
        #  ...

        json_data = Column(json_type)

.. seealso::

      :ref:`mutable_toplevel` 

比较运算
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :class:`.TypeDecorator` ` JSONEncodedDict``的类型，我们需要在使用此运算符之前使用 :func:`.cast` 或
 :func:`type_coerce` 来将列转换为文本形式。::

    from sqlalchemy import type_coerce, String

    stmt = select(my_table).where(type_coerce(my_table.c.json_data, String).like("%foo%"))

  :class:`.TypeDecorator` .TypeDecorator.coerce_compared_value`  方法来定义新运算符::

    from sqlalchemy.sql import operators
    from sqlalchemy import String


    class JSONEncodedDict(TypeDecorator):
        impl = VARCHAR

        cache_ok = True

        def coerce_compared_value(self, op, value):
            if op in (operators.like_op, operators.not_like_op):
                return String()
            else:
                return self

        def process_bind_param(self, value, dialect):
            if value is not None:
                value = json.dumps(value)

            return value

        def process_result_value(self, value, dialect):
            if value is not None:
                value = json.loads(value)
            return value

以上只是处理“LIKE”操作符的方法之一。其他应用程序可能希望为JSON对象引发``NotImplementedError``的操作符（如“LIKE”），而不是自动将其强制类型为文本。

.. _types_sql_value_processing:

应用SQL级别的Bind/Result处理
-----------------------------------------

如在 :ref:`types_typedecorator` 一节中所述，当将参数发送到语句时和从数据库加载结果行时，SQLAlchemy允许调用Python函数以应用转换。还可以定义SQL级别的转换。这样做的原因是在关系数据库中包含了一系列特定于应用程序和持久性格式之间转换的功能而只存在于数据库服务器中。

任何: class: 'TypeEngine'，: class :。'UserDefinedType'或 : class:`.TypeDecorator`子类均可包括  :meth:`.TypeEngine.bind_expression`   和 / 或  :meth:` .TypeEngine.column_expression`   的实现，如果定义为返回非“None”值，则应返回: class:`_expression.ColumnElement ` 为将注入到SQL语句中，包围绑定参数或列表达式的表达式格式。例如，要构建一个“Geometry”类型，该类型将对所有传出值应用PostGIS函数“ST_GeomFromText”，并对所有传入数据应用函数“ST_AsText”，我们可以创建一个  :class:`.UserDefinedType` ~.sqlalchemy.sql.expression.func` 结合使用::

    from sqlalchemy import func
    from sqlalchemy.types import UserDefinedType


    class Geometry(UserDefinedType):
        def get_col_spec(self):
            return "GEOMETRY"

        def bind_expression(self, bindvalue):
            return func.ST_GeomFromText(bindvalue, type_=self)

        def column_expression(self, col):
            return func.ST_AsText(col, type_=self)

我们可以将“Geometry”类型应用于:schema:Table元数据中，并在:func :`_expression.select`中使用它::

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

生成的SQL将嵌入两个函数。 ST_AsText应用于columns条款，以便将ST_GeomFromText在将返回值传递到结果集之前运行：

.. sourcecode:: sql

    SELECT geometry.geom_id, ST_AsText(geometry.geom_data) AS geom_data_1
    FROM geometry
    WHERE geometry.geom_data = ST_GeomFromText(:geom_data_2)

通过  :meth:`.Operators.op`   中使用自定义SQL进行比较操作,参数  :paramref:` .Operators.op.is_comparison`  标志应设置为“True”::

    class MyInt(Integer):
        class comparator_factory(Integer.Comparator):
            def is_frobnozzled(self, other):
                return self.op("--is_frobnozzled->", is_comparison=True)(other)

.. seealso::

     :meth:`.Operators.op` 

     :attr:`.TypeEngine.comparator_factory` 

创建新类型
------------------

  :class:`.UserDefinedType` .TypeDecorator` 。

.. autoclass:: UserDefinedType
   :members:

   .. autoattribute:: cache_ok

.. _custom_and_decorated_types_reflection:


处理自定义类型和反射
-----------------------------------------

请注意，具有附加Python行为的数据库类型，包括基于
 :class:`.TypeDecorator` 和其他用户定义的数据类型的类型，
在数据库模式中没有任何表现形式。当使用数据库时
在描述 :ref:`metadata_reflection` 处的功能时，SQLAlchemy使用一个固定的映射，
其将由数据库服务器报告的数据类型信息与SQLAlchemy数据类型对象相关联。
例如，如果我们在PostgreSQL模式中查看特定数据库列的定义，我们可能会收回字符串“VARCHAR”。
SQLAlchemy的PostgreSQL方言具有硬编码映射，将字符串名称“VARCHAR”与SQLAlchemy  :class:`.VARCHAR` ”这样的语句时，  :class:` _schema.Column` .VARCHAR`。

这的含义是，如果表对象使用无法直接对应于数据库本机类型名称的类型对象，那么如果我们使用反映创建一个新的  :class:`_schema.Table`  _schema.MetaData` 集合中的此数据库表，不能包含此数据类型。例如：：

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

上面，我们使用了  :class:`.PickleType` ，它是一个  :class:` .TypeDecorator` ，它基于  :class:`.LargeBinary` .PickleType` 。

如果我们查看“my_table.c.data.type”的类型，因为这是由我们直接创建的Python对象，它是：class:`.PickleType`::

    >>> my_table.c.data.type
    PickleType()

但是，如果我们使用反射创建另一个  :class:`_schema.Table` .PickleType` 的使用不会反映在我们创建的SQLite数据库中。相反，我们得到了  :class:`.BLOB` ：

.. sourcecode:: pycon+sql

    >>> metadata_two = MetaData()
    >>> my_reflected_table = Table("my_table", metadata_two, autoload_with=engine)
    {execsql}INFO sqlalchemy.engine.base.Engine PRAGMA main.table_info("my_table")
    INFO sqlalchemy.engine.base.Engine ()
    DEBUG sqlalchemy.engine.base.Engine Col ('cid', 'name', 'type', 'notnull', 'dflt_value', 'pk')

通常，当一个应用程序使用自定义类型定义了显式的   :class:`_schema.Table`  元数据时，就不需要使用表反射，因为所需的   :class:` _schema.Table`  元数据已经存在。然而，对于应用程序或它们的组合需要使用既包括自定义的 Python 层次结构数据类型，又设置其   :class:`_schema.Column`  对象为从数据库反射的   :class:` _schema.Table`  对象，仍需要展示自定义数据类型的额外 Python 行为的情况，需要采取额外措施。

最直接的方法是覆盖特定的列，如   :ref:`reflection_overriding_columns`  中所述。在这种技术中，我们使用反射与显式   :class:` _schema.Column`  对象相结合，对于那些我们想要使用自定义或装饰数据类型的列，我们使用他们来覆盖反射的对象：

    >>> metadata_three = MetaData()
    >>> my_reflected_table = Table(
    ...     "my_table",
    ...     metadata_three,
    ...     Column("data", PickleType),
    ...     autoload_with=engine,
    ... )

上述的 `my_reflected_table` 对象是反射的，会从 SQLite 数据库中加载 "id" 列的定义。但是对于 "data" 列，我们已经用包含我们想要的 Python 数据类型   :class:`.PickleType`  的   :class:` _schema.Column`  显式定义来覆盖了反射的对象。反射过程将保持此   :class:`_schema.Column`  对象不变：

    >>> my_reflected_table.c.data.type
    PickleType()

从数据库本地类型对象转换为自定义数据类型的更复杂的方法是使用  :meth:`.DDLEvents.column_reflect`  事件处理程序。如果我们知道我们希望所有的   :class:` .BLOB`  数据类型实际上都是   :class:`.PickleType` ，我们可以设置一条规则::

    from sqlalchemy import BLOB
    from sqlalchemy import event
    from sqlalchemy import PickleType
    from sqlalchemy import Table


    @event.listens_for(Table, "column_reflect")
    def _setup_pickletype(inspector, table, column_info):
        if isinstance(column_info["type"], BLOB):
            column_info["type"] = PickleType()

在反射任何包含   :class:`.BLOB`  数据类型列的   :class:` _schema.Table`  时，如果在应用程序中调用了上述代码 *之前* （同时注意在应用程序中仅调用 **一次**，因为这是一个全局规则），所得到的数据类型将在   :class:`_schema.Column`  对象中以   :class:` .PickleType`  存储。

在实践中，上述基于事件的方法可能有附加规则，以便仅影响那些数据类型很重要的列，例如表名和可能的列名的查找表，或其他启发式规则，以便准确确定哪些列应该使用 Python 数据类型来建立。