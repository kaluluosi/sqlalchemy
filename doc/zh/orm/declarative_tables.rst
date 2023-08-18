.. _orm_declarative_table_config_toplevel:

=============================================
使用Declarative配置Table
=============================================

在   :ref:`orm_declarative_mapping`  中介绍了Declarative风格包含生成映射的   :class:` _schema.Table`  对象的功能或直接容纳   :class:`_schema.Table`  或其他   :class:` _sql.FromClause`  对象。

下面的示例假设Declarative基类为::

    from sqlalchemy.orm import DeclarativeBase

    class Base(DeclarativeBase):
        pass

所有接下来的示例都展示了从上面继承 ``Base`` 类。 此外，   :ref:`orm_declarative_decorator`  中引入的装饰器样式也完全支持所有以下示例，以及通过   :func:` _orm.declarative_base`  生成的基类形式的Declarative Base。

.. _orm_declarative_table:

使用 ``mapped_column()`` 的Declarative Table
--------------------------------------------

使用Declarative时，要映射的类的主体在大多数情况下包括一个名为 ``__tablename__`` 的属性，该属性指示应在映射期间生成一个   :class:`_schema.Table`  的字符串名称。 然后在类主体内使用   :func:` _orm.mapped_column`  构造，该构造包含ORM特定的其他配置能力，这些能力在普通的   :class:`_schema.Column`  类中不可用，以指示表中的列。

下面的示例演示了在Declarative映射中使用此构造的最基本的用法::


    from sqlalchemy import Integer, String
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import mapped_column

    class Base(DeclarativeBase):
        pass

    class User(Base):
        __tablename__ = "user"

        id = mapped_column(Integer, primary_key=True)
        name = mapped_column(String(50), nullable=False)
        fullname = mapped_column(String)
        nickname = mapped_column(String(30))


上面，   :func:`_orm.mapped_column`  构造被放置在类定义内的类级别属性中。在声明类的时，Declarative映射过程将针对与Declarative“Base”相关联的   :class:` _schema.MetaData`  集合生成一个新的   :class:`_schema.Table`  对象；然后将使用每个   :func:` _orm.mapped_column`  来在此过程中生成一个   :class:`_schema.Column`  对象，该对象将成为该   :class:` _schema.Table`  对象的  :attr:`.schema.Table.columns`  集合的一部分。

在上面的示例中，Declarative将构建一个等价于以下内容的   :class:`_schema.Table`  构造::

    # equivalent Table object produced
    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String()),
        Column("nickname", String(30)),
    )

当上面的 ``User`` 类被映射时，此   :class:`_schema.Table`  对象可以通过 ` `__table__`` 属性直接访问；有关详细信息，请参见   :ref:`orm_declarative_metadata` 。

.. 侧边栏::  ``mapped_column()`` 取代了使用 ``Column()``

  1.x SQLAlchemy的用户将注意到   :func:`_orm.mapped_column`  构造的使用是新的，这是 SQLAlchemy 2.0 系列的新增功能。上述为 ORM 特定的构造旨在首要目标是 Declarative 映射内使用增加 ORM-特定的方便功能，例如此构造在该构造中建立  :paramref:` _orm.mapped_column.deferred`  的能力，对被 Mypy_ 和 Pylance_ 等打印工具表示某些属性在运行时的实现也是有效的。正如下文中所述，它还是在新的基于注释的配置样式的前沿，由 SQLAlchemy 2.0 引入。

  遗留代码的用户应该注意，   :class:`_schema.Column`  的形式始终在 Declarative 中按照以往的方式工作。还可以在属性上逐个基础修改地混合使用不同形式的特性映射，因此可以以任何节奏迁移到新形式。请参见   :ref:` whatsnew_20_orm_declarative_typing`  部分，其中提供了将 Declarative 模型迁移到新形式的逐步指南。

  :func:`_orm.mapped_column`  构造接受   :class:` _schema.Column`  构造接受的所有参数，以及其他特定于ORM的参数。  :paramref:`_orm.mapped_column.__name`  字段表示数据库列的名称，通常被省略，因为Declarative过程将利用所给予构造的属性名称，并将其分配为列名称（在上面的示例中，这是指名称“id”，“name”，“fullname”，“nickname”）。指定替代  :paramref:` _orm.mapped_column.__name`  也是有效的，   :class:`_schema.Column`  将在 SQL 和 DDL 语句中使用给定名称，而 ` `User`` 映射类将继续允许访问使用给定属性名称的属性，与列本身的名称无关（更多内容请参见   :ref:`mapper_column_distinct_names` ）。

.. tip::

      :func:`_orm.mapped_column`  构造 **仅在Declarative类映射内有效**。在构建   :class:` _schema.Table`  对象时，无论是使用Core还是使用   :ref:`imperative table <orm_imperative_table_configuration>`  配置，都需要   :class:` _schema.Column`  构造来指示存在数据库列。

.. seealso::

      :ref:`mapping_columns_toplevel`  - 包含有关影响   :class:` _orm.Mapper`  如何解释传入的   :class:`.Column`  对象的其他注释。

.. _orm_declarative_mapped_column:

在类型注释中使用注释式的Declarative Table (``mapped_column()`` 的类型注释形式)
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_orm.mapped_column`  构造能够从关联到在用于声明Declarative映射类的Python缺省  :attr:` .Mapped`  类型注释中的类型注释  :pep:`484`  中派生其列配置信息。如果使用，则这些类型注释**必须**存在于名为   :class:` _orm.Mapped`  的特殊的 SQLAlchemy 类型内，该类型是一个通用类型，然后表示其中一个特定的Python类型。

下面演示了从上一节中进行映射，添加了对使用   :class:`_orm.Mapped`  的示例::

    from typing import Optional

    from sqlalchemy import String
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(50))
        fullname: Mapped[Optional[str]]
        nickname: Mapped[Optional[str]] = mapped_column(String(30))


上面的示例中，Declarative在每个类属性进行处理时，如果存在，则每个   :func:`_orm.mapped_column`  将从相应的左侧   :class:` _orm.Mapped`  类型注释派生出额外的参数。此外，当遇到没有分配任何值的   :class:`_orm.Mapped`  类型注释的情况下，Declarative将在此类形式的情况下隐式生成一个空的   :func:` _orm.mapped_column`  指令，进而从   :class:`_orm.Mapped`  注释中派生其配置。

.. _orm_declarative_mapped_column_nullability:

``mapped_column()`` 从 ``Mapped`` 注释中派生类型和可空性
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

从   :class:`_orm.Mapped`  注释导出的   :func:` _orm.mapped_column`  得出的两个特性是：

* **datatype** - 给定内部的   :class:`_orm.Mapped`  中的 Python 类型，包含在 ` `typing.Optional`` 中的构造（如果存在）与   :class:`_sqltypes.TypeEngine`  子类相关联，例如  :class:` .Integer` ，   :class:`.String` ，   :class:` .DateTime`  或   :class:`.Uuid`  等常见类型。

  数据类型是根据 Python 类型到 SQLAlchemy 数据类型的字典确定的。此字典是完全可定制的，如下一节   :ref:`orm_declarative_mapped_column_type_map`  所述。 默认类型映射实现如下面的代码示例：：

      from typing import Any
      from typing import Dict
      from typing import Type

      import datetime
      import decimal
      import uuid

      from sqlalchemy import types

      # 基于 Mapped[] 注释导出类型的默认映射，此构造方式仅适用于 Declarative 映射
      type_map: Dict[Type[Any], TypeEngine[Any]] = {
          bool: types.Boolean(),
          bytes: types.LargeBinary(),
          datetime.date: types.Date(),
          datetime.datetime: types.DateTime(),
          datetime.time: types.Time(),
          datetime.timedelta: types.Interval(),
          decimal.Decimal: types.Numeric(),
          float: types.Float(),
          int: types.Integer(),
          str: types.String(),
          uuid.UUID: types.Uuid(),
      }

  如果   :func:`_orm.mapped_column`  构造指示传递给  :paramref:` _orm.mapped_column.__type`  参数的显式类型，则忽略给定的 Python 类型。

* **nullability** - 在   :class:`_schema.Column`  中，   :func:` _orm.mapped_column`  构造首先使用  :paramref:`_orm.mapped_column.nullable`  参数来指示其   :class:` _schema.Column`  为 ``NULL`` 或 ``NOT NULL``。此外，如果出现  :paramref:`_orm.mapped_column.primary_key`  参数并设置为 ` `True``，则还将意味着该列应为 ``NOT NULL``。

  如果缺少**这两个**参数，则   :class:`_orm.Mapped`  类型注释中存在 ` `typing.Optional[]`` 的存在将用于确定可空性，其中 ``typing.Optional[]`` 表示 ``NULL``，而不包含 ``typing.Optional[]`` 则表示 ``NOT NULL``。如果根本没有出现 ``Mapped[]`` 注释，并且没有  :paramref:`_orm.mapped_column.nullable`  或  :paramref:` _orm.mapped_column.primary_key`  参数，则使用 SQLAlchemy 的   :class:`_schema.Column`  的通常默认 ` `NULL``。

  在下面的示例中， ``id`` 和 ``data`` 列将为 ``NOT NULL``，而 ``additional_info`` 列将是 ``NULL``；

      from typing import Optional

      from sqlalchemy.orm import DeclarativeBase
      from sqlalchemy.orm import Mapped
      from sqlalchemy.orm import mapped_column


      class Base(DeclarativeBase):
          pass


      class SomeClass(Base):
          __tablename__ = "some_table"

          # primary_key=True，因此将为 NOT NULL
          id: Mapped[int] = mapped_column(primary_key=True)

          # not Optional[]，因此将为 NOT NULL
          data: Mapped[str]

          # Optional[]，因此将为 NULL
          additional_info: Mapped[Optional[str]]


同时，还可以拥有   :func:`_orm.mapped_column`  ，其可空性与注释所表示的可能**不同**。例如，ORM映射属性可以在 Python 代码中标记为允许 ` `None``，因为它在首次创建和填充对象时使用，但最终该值将写入在模式级别上为 ``NOT NULL`` 的数据库列中。当  :paramref:`_orm.mapped_column.nullable`  参数存在时，始终会优先使用：

    class SomeClass(Base):
        # ...

        # pep-484类型将为Optional，但是值在数据库中是 NOT NULL
        created_at: Mapped[Optional[timestamp]] = mapped_column(nullable=False)

同样，对于必须为非 None 而写入数据库列的 ORM 映射属性， 则列如果需要为 NULL，即使数据在 Python 中是非 None 的，  :paramref:`_orm.mapped_column.nullable`  也必须设置为 ` `True``：

    class SomeClass(Base):
        # ...

        # 将为 NULL 的 String()，但类型检查器将不会在预期此属性为None时给出提示
        data: Mapped[str] = mapped_column(nullable=True)

.. _orm_declarative_mapped_column_type_map:

自定义类型映射
~~~~~~~~~~~~~~~~~

在以前的章节中描述的 Python 类型与 SQLAlchemy   :class:`_types.TypeEngine`  类型之间的映射默认为硬编码的字典，在 ` `sqlalchemy.sql.sqltypes`` 模块中提供。但是，Declarative映射过程的协调器   :class:`_orm.registry`  对象将在首次使用时首先查看一个用户定义的本地字典，该字典可能会传递为构建   :class:` _orm.registry`  时的  :paramref:`_orm.registry.type_annotation_map`  参数，并且可以与   :class:` _orm.DeclarativeBase`  超类相关联。

例如，如果我们希望将 Python ``int`` 映射到 ``_sqltypes.BIGINT`` 数据类型，将 Python ``datetime.datetime`` 映射到带有 ``timezone=True`` 的 ``_sqltypes.TIMESTAMP`` 数据类型，并且仅在 Microsoft SQL Server 上使用 ``_sqltypes.NVARCHAR`` 数据类型时使用 Python ``str``，则可对 registry 和 Declarative base 进行配置::

    import datetime

    from sqlalchemy import BIGINT, Integer, NVARCHAR, String, TIMESTAMP
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped, mapped_column, registry


    class Base(DeclarativeBase):
        type_annotation_map = {
            int: BIGINT,
            datetime.datetime: TIMESTAMP(timezone=True),
            str: String().with_variant(NVARCHAR, "mssql"),
        }


    class SomeClass(Base):
        __tablename__ = "some_table"

        short_name: Mapped[str_30] = mapped_column(primary_key=True)
        long_name: Mapped[str_50]
        num_value: Mapped[num_12_4]
        short_num_value: Mapped[num_6_2]


在上面的示例中，Python 类型被作为键传递到  :paramref:`_orm.registry.type_annotation_map`  字典中， link the base  :class:` _types.TypeEngine`来升级数据类型的设置。 

@RunWith：_::`大型二进制`，   :class:`datetime.datetime`   可配置的参数，其中较少的一部分是在 ` `_sqltypes`` 模块中定义。 因此，使用  :paramref:`_orm.registry.type_annotation_map`  的便捷性在于，您可以根据需要链接类似的类型，例如将 Python ` `int`` 映射到 ``_sqltypes.BIGINT`` 或 ``_sqltypes.INTEGER`` 或其他兼容的 SQL 数据类型::

    from sqlalchemy import BIGINT, INTEGER
    
    class Base(DeclarativeBase):
        type_annotation_map = {
            int: BIGINT,
            int: INTEGER,
        }

如果上面的示例未满足您的要求，您也可以像下面的示例那样声明自己的类型映射字典，以连接适合您应用以及要使用的数据库的类型::

    from sqlalchemy.types import TypeEngine, TypeDecorator
    from typing import Any, Dict, Type

    type_map: Dict[Type[Any], TypeEngine[Any]] = {
        CustomDataType: TypeDecorator(custom_type_str),
        int: INTEGER,
        str: String(50),
        decimal.Decimal: Numeric(22, 4),
        datetime.datetime: TIMESTAMP(),
    }

    class CustomClass(Base):
        pass


    custom_registry = registry(type_annotation_map=type_map)

在本示例中，我们连接了四个不同的数据类型（其中一个是自定义类型）以及一个自定义数据类型到我们想要使用的SQLAlchemy数据类型的映射。

.. _orm_declarative_mapped_column_type_map_pep593:

将多个类型配置映射到 Python 类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

由于类型注释可与任何使用 Python 内置 ``enum.Enum`` 的用户定义 Python 类型作为键相关联，因此也可以使用 ``typing.Literal`` 类型字面值作为键。 

下面是一个示例，用于到 ``Literal`` 类型作为键，通过数据类型控制字符串的输入内容::


    from typing import Literal

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    Status = Literal["pending", "received", "completed"]


    class SomeClass(Base):
        __tablename__ = "some_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        status: Mapped[Status]


在上面的示例中，“SomeClass.status” 映射属性将与数据类型为 ``Enum(Status)`` 的列连接。 我们可以在使用 PostgreSQL 数据库时看到这一点，如下所示：

.. sourcecode:: sql

  CREATE TYPE status AS ENUM ('PENDING', 'RECEIVED', 'COMPLETED')

  CREATE TABLE some_table (
    id SERIAL NOT NULL,
    status status NOT NULL,
    PRIMARY KEY (id)
  )


同样地，也可以使用  :pep:`593`  的 ` `Annotated`` 构造函数及 Python 的 ``enum.Enum`` 类型定义来添加元数据到类型中，实现对 ORM 映射的自定义控制。以下示例使用自定义枚举类型 ``Status`` 作为映射类型：：

    import enum

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class Status(enum.Enum):
        PENDING = "pending"
        RECEIVED = "received"
        COMPLETED = "completed"


    class SomeClass(Base):
        __tablename__ = "some_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        status: Mapped[Status]

在上面的示例中，映射属性 ``SomeClass.status`` 将链接到具有 ``Enum(Status)`` 数据类型的   :class:`.Column` 。我们可以看到下面的 PostgreSQL 数据库中，` `CREATE TABLE`` 输出：：

.. sourcecode:: sql

  CREATE TYPE status AS ENUM ('PENDING', 'RECEIVED', 'COMPLETED')

  CREATE TABLE some_table (
    id SERIAL NOT NULL,
    status status NOT NULL,
    PRIMARY KEY (id)
  )


.. "Literal"的使用，对于 ``typing_extensions`` 未安装的版本请自行查找方法。

.. versionadded:: 2.0.1, 支持 ``Literal`` 类型字面值协议，可以使用 ``typing.Literal`` 定义类参数的值。

.. _orm_declarative_mapped_column_pep593:

将整个列声明映射到Python类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

前面的部分演示了如何将   :func:`_orm.mapped_column`  连接到定义在 ` `Annotated`` 中的特定列和函数中。使用此形式，我们可以定义不仅适用于Python类型的相似SQL数据类型拼接，还可以设置许多其他参数，例如时间戳、默认值和约束等，其中许多设置是预先定义的在声明中的列所具有的。我们可以将这些配置组成可重用的   :func:`_orm.mapped_column`  构造函数，然后直接从 ` `Annotated`` 实例中提取它们，在您的类定义中可以多次使用。

ORM模型通常具有所有映射类都通用的主键样式。此外，可能存在相似的列配置，例如具有默认值的时间戳和其他预先设置的大小和配置的字段。我们可以将这些配置组合成   :func:`_orm.mapped_column`  实例，然后将实例直接打包到 ` `Annotated`` 实例中，以便在任意数量的类声明中重复使用此配置。当以此方式提供 ``Annotated`` 对象时，Declarative 会解压缩 ``Annotated`` 对象，跳过不适用于 SQLAlchemy 的任何其他指令，并仅搜索 SQLAlchemy ORM 构造。

下面的示例演示了用于此类用途的各种预配置的字段类型，其中我们定义了 ``intpk`` 以代表一个整数类型的主键列， ``timestamp`` 代表一个   :class:`.DateTime`  类型，将使用 ` `CURRENT_TIMESTAMP`` 作为 DDL 级别的列默认值，以及 ``required_name`` ，它是一个长度为 30 的   :class:`.String`  且为 ` `NOT NULL`` 的字段::

    import datetime

    from typing_extensions import Annotated

    from sqlalchemy import func
    from sqlalchemy import String
    from sqlalchemy.orm import mapped_column


    intpk = Annotated[int, mapped_column(primary_key=True)]
    timestamp = Annotated[
        datetime.datetime,
        mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
    ]
    required_name = Annotated[str, mapped_column(String(30), nullable=False)]

在上面的示例中， ``Annotated`` 对象可以直接在   :class:`_orm.Mapped`  中使用，其中预配置的   :func:` _orm.mapped_column`  构造将从 ``Annotated`` 对象中提取并复制到将针对每个属性特定的新实例中::

    class Base(DeclarativeBase):
        pass

    class SomeClass(Base):
        __tablename__ = "some_table"

        id: Mapped[intpk]
        name: Mapped[required_name]
        created_at: Mapped[timestamp]

下面是用于以上映射创建的 ``CREATE TABLE`` 语句：

.. sourcecode:: sql

    >>> from sqlalchemy.schema import CreateTable
    >>> print(CreateTable(SomeClass.__table__))
    {printsql}CREATE TABLE some_table (
      id INTEGER NOT NULL,
      name VARCHAR(30) NOT NULL,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP NOT NULL,
      PRIMARY KEY (id)
    )

.. 注意：在 ORM 操作的其它构造函数中，例如   :func:`_orm.relationship`  和   :func:` _orm.composite`  中尚未实现 ``Annotated`` 对象的前一节所述的特性。虽然这个功能在理论上是可行的，但目前尝试在此类中使用 ``Annotated`` 来指示其他参数将在运行时引发 ``NotImplementedError`` 异常，但可能会在将来的版本中实现。``Enum.Enum`` Python类型以及 ``typing.Literal`` 类型都可以映射到SQLAlchemy   :class:`.Enum`  SQL 类型，使用一个特殊形式指示给   :class:` .Enum`  数据类型，它应该针对任何枚举类型自动配置自己。这个默认情况下隐含的配置可以显式地指定如下：

    import enum
    import typing

    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        type_annotation_map = {
            enum.Enum: sqlalchemy.Enum(enum.Enum),
            typing.Literal: sqlalchemy.Enum(enum.Enum),
        }

在 Declarative 中解析逻辑能够解析 ``Enum.Enum`` 子类型以及 ``typing.Literal`` 的实例，来匹配  :paramref:`_orm.registry.type_annotation_map`  字典中的 ` `Enum.Enum`` 或 ``typing.Literal`` 条目。   :class:`.Enum`  SQL 类型然后知道如何生成一个配置的版本，包括默认字符串长度。如果传递了一个不仅包含字符串值的 ` `typing.Literal``，则会引发具有说明性的错误信息。

本机 Enums 和命名
+++++++++++++++++++++++++

  :paramref:`.sqltypes.Enum.native_enum`   参数指的是是否应该创建所谓的 “本机” 枚举。在 MySQL/MariaDB 中是 ` `ENUM`` 数据类型，在 PostgreSQL 中是由 ``CREATE TYPE`` 创建的新的 ``TYPE`` 对象，或者是 “非本机” 枚举，这意味着将使用 ``VARCHAR`` 创建数据类型。对于除 MySQL/MariaDB 或 PostgreSQL 之外的后端，将在所有情况下使用 ``VARCHAR`` (第三方方言可能具有其自己的行为）。

由于 PostgreSQL 的 ``CREATE TYPE`` 要求为要创建的类型指定显式名称，因此当使用隐式生成的   :class:`.sqltypes.Enum`  且未在映射中指定显式的   :class:` .sqltypes.Enum`  数据类型时，存在特殊的回滚逻辑：

1. 如果   :class:`.sqltypes.Enum`  与 ` `enum.Enum`` 对象相关联，则  :paramref:`.sqltypes.Enum.native_enum`  参数默认为 ` `True``，并且枚举的名称将来自 ``enum.Enum`` 数据类型的名称。PostgreSQL 后端将默认假定 ``CREATE TYPE`` 以此名称。

2. 如果   :class:`.sqltypes.Enum`  链接到 ` `typing.Literal`` 对象，则  :paramref:`.sqltypes.Enum.native_enum`  参数默认为 ` `False``，不生成名称，且假定使用 ``VARCHAR``。

要将 ``typing.Literal`` 与 PostgreSQL ``CREATE TYPE`` 类型一起使用，必须使用显式   :class:`.sqltypes.Enum` ，无论是在类型映射中：

    import enum
    import typing

    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase

    Status = Literal["pending", "received", "completed"]


    class Base(DeclarativeBase):
        type_annotation_map = {
            Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
        }

或者是在   :func:`_orm.mapped_column`  中：

    import enum
    import typing

    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase

    Status = Literal["pending", "received", "completed"]


    class Base(DeclarativeBase):
        pass


    class SomeClass(Base):
        __tablename__ = "some_table"

        id: Mapped[int] = mapped_column(primary_key=True)
        status: Mapped[Status] = mapped_column(
            sqlalchemy.Enum("pending", "received", "completed", name="status_enum")
        )

更改默认枚举的配置
++++++++++++++++++++++++++

为了修改隐式生成的   :class:`.Enum.Enum`  数据类型的固定配置，请在  :paramref:` _orm.registry.type_annotation_map`  中指定额外的条目，以指示其他参数。例如，要无条件地使用 “非本机枚举”，可以将  :paramref:`.Enum.native_enum`  参数设置为 ` `False``：

    import enum
    import typing
    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        type_annotation_map = {
            enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
            typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
        }

.. versionchanged:: 2.0.1 实现了支持在建立  :paramref:`_orm.registry.type_annotation_map`  时重写  :paramref:` _sqltypes.Enum.native_enum`  等参数的功能。先前，这个功能无法使用。

要为特定的 ``Enum.Enum`` 子类型使用特定配置，例如在使用上面的 ``Status`` 数据类型时将字符串长度设置为 50：

    import enum
    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase


    class Status(enum.Enum):
        PENDING = "pending"
        RECEIVED = "received"
        COMPLETED = "completed"


    class Base(DeclarativeBase):
        type_annotation_map = {
            Status: sqlalchemy.Enum(Status, length=50, native_enum=False)
        }

链接特定的 ``Enum.Enum`` 或 ``typing.Literal`` 到其他数据类型
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

上面的示例演示了   :class:`_sqltypes.Enum`  如何自动配置适用于 ` `Enum.Enum`` 或 ``typing.Literal`` 类型对象的参数/属性。对于特定类型应链接到其他类型的用例中，这些特定类型也可以放置在类型映射中。在下面的示例中，包含非字符串类型的 ``Literal[]`` 条目将链接到   :class:`_sqltypes.JSON`  数据类型：

    from typing import Literal

    from sqlalchemy import JSON
    from sqlalchemy.orm import DeclarativeBase

    my_literal = Literal[0, 1, True, False, "true", "false"]


    class Base(DeclarativeBase):
        type_annotation_map = {my_literal: JSON}

在以上配置中， ``my_literal`` 数据类型将解析为一个   :class:`._sqltypes.JSON`  实例。其他 ` `Literal`` 变体将继续解析为   :class:`_sqltypes.Enum`  数据类型。


``mapped_column()`` 中的 dataclass 特性
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :func:`_orm.mapped_column`  结构集成了 SQLAlchemy 的 “原生数据类” 特性，如在
  :ref:`orm_declarative_native_dataclasses`  中所述。请参阅该部分，了解   :func:` _orm.mapped_column`  支持的其他指令。

.. _orm_declarative_metadata:

访问表和元数据
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

装饰器声明的类将始终包括一个名为 ``__table__`` 的属性；使用上述使用 ``__tablename__`` 的配置后，声明过程将通过 ``__table__`` 属性使   :class:`_schema.Table`  可用：

    # 访问 Table
    user_table = User.__table__

上述表最终与  :attr:`_orm.Mapper.local_table`  属性对应，我们可以通过   :ref:` 运行时检测系统 <inspection_toplevel>`  来查看：

    from sqlalchemy import inspect

    user_table = inspect(User).local_table

在 ddl 操作(例如 CREATE) 以及与迁移工具（例如 Alembic）一起使用时，与 Declarative 注册基类相关联的   :class:`_schema.MetaData`  集合通常是必需的。该对象可以通过   :class:` _orm.registry`  的 ``.metadata`` 属性以及 Declarative 基类进行访问。以下是一个小脚本，我们可能希望使用它来针对 SQLite 数据库发出 CREATE 所有表的命令：

    engine = create_engine("sqlite://")

    Base.metadata.create_all(engine)

.. _orm_declarative_table_configuration:

声明式表配置
^^^^^^^^^^^^^^^^^^^^^^^^^^

使用 Declarative 表配置时，提供应提供给   :class:`_schema.Table`  构造函数的附加参数应在 ` `__table_args__`` 声明类属性中指定。

该属性接受按位置排列的参数和关键字参数，这些通常发送到   :class:`_schema.Table`  构造函数。
可以使用字典指定属性：

    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = {"mysql_engine": "InnoDB"}

或者，使用每个参数作为位置参数（通常是约束条件）的元组：

    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = (
            ForeignKeyConstraint(["id"], ["remote_table.id"]),
            UniqueConstraint("foo"),
        )

通过将字典指定为最后一个参数，可以使用关键字参数：

    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = (
            ForeignKeyConstraint(["id"], ["remote_table.id"]),
            UniqueConstraint("foo"),
            {"autoload": True},
        )

一个类还可以使用   :func:`_orm.declared_attr`  方法装饰器以动态样式指定 ` `__table_args__`` 声明属性，以及 ``__tablename__`` 属性。请参阅   :ref:`orm_mixins_toplevel`  了解详情。

.. _orm_declarative_table_schema_name:

使用 Declarative Table 的显式模式名称
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

  :class:`_schema.Table`  中描述的模式名称   :ref:` schema_table_schema_name`  被应用于一个   :class:`_schema.Table` ，使用  :paramref:` _schema.Table.schema`  参数。当使用 Declarative 时，此选项应像任何其他选项一样在 ``__table_args__`` 字典中传递：

    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = {"schema": "some_schema"}

.. seealso::

      :ref:`schema_table_schema_name`  - 在   :ref:` metadata_toplevel`  文档中。

.. _orm_declarative_table_column_naming:

为映射表列显式命名
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

上述示例演示了如何使用   :func:`_orm.mapped_column`  为生成的   :class:` _schema.Column`  对象提供与它们映射的属性名称不同的在 SQL 中表达的名称。

在使用 Imperative Table 配置时，我们已经有了现有的   :class:`_schema.Column`  对象。为了将这些   :class:` _schema.Column`  对象从与它们绑定的属性名称映射到备用名称，我们可以直接将   :class:`_schema.Column`  赋值给所需的属性：

    user_table = Table(
        "user",
        Base.metadata,
        Column("user_id", Integer, primary_key=True),
        Column("user_name", String),
    )


    class User(Base):
        __table__ = user_table

        id = user_table.c.user_id
        name = user_table.c.user_name

上述 ``User`` 映射将通过 ``User.id`` 和 ``User.name`` 属性引用 ``"user_id"`` 和 ``"user_name"`` 列，在和在   :ref:`orm_declarative_table_column_naming`  中演示的方式相同。

上述映射的一个注意事项是，在使用  :pep:`484`  类型提示工具时，将无法正确地对直接的行链接到   :class:` _schema.Column` 。解决此问题的策略是在   :func:`_orm.column_property`  中应用   :class:` _schema.Column`  对象；虽然   :class:`_orm.Mapper`  已经自动为其内部使用生成了此属性对象，但是通过在类声明中命名它，类型提示工具将能够将属性与   :class:` _orm.Mapped`  注释相匹配：

    from sqlalchemy.orm import column_property
    from sqlalchemy.orm import Mapped


    class User(Base):
        __table__ = user_table

        id: Mapped[int] = column_property(user_table.c.user_id)
        name: Mapped[str] = column_property(user_table.c.user_name)

.. seealso::

      :ref:`orm_declarative_table_column_naming`  - 适用于 Declarative Table

.. _orm_imperative_table_configuration:

具有 Imperative Table 的 Declarative （俗称混合 Declarative）
---------------------------------------------------------------

声明式映射可以映射到一个通过反射进入的   :class:`_schema.Table`  对象系列，这些对象使用在   :ref:` metadata_reflection`  中描述的反射过程进行了内省。

将一个类映射到从数据库反射出来的表，简单的示例如下所示，使用混合声明式映射：

    from sqlalchemy import create_engine
    from sqlalchemy import Table
    from sqlalchemy.orm import DeclarativeBase

    engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")


    class Base(DeclarativeBase):
        pass


    class MyClass(Base):
        __table__ = Table(
            "mytable",
            Base.metadata,
            autoload_with=engine,
        )

为了将多个表映射到一个   :class:`.MetaData` ，可以使用  :meth:` .MetaData.reflect`  方法一次反射完整的   :class:`.Table`  对象集，然后从   :class:` .MetaData`  引用它们：

    from sqlalchemy import create_engine
    from sqlalchemy import Table
    from sqlalchemy.orm import DeclarativeBase

    engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")


    class Base(DeclarativeBase):
        pass


    Base.metadata.reflect(engine)


    class MyClass(Base):
        __table__ = Base.metadata.tables["mytable"]

上述方法中的一个提示是，除非表的元数据真正地被反射出来，否则无法声明映射类。这要求在声明应用程序类时存在数据库连接源；通常的情况是，当应用程序的模块被导入时，类被声明，但数据库连接需要应用程序开始运行代码才会可用，以便使用配置信息并创建引擎。目前有两种方法可以解决这个问题，在下面的两个部分中描述。

.. _orm_declarative_reflected_deferred_reflection:

使用 DeferredReflection（延迟反射）
++++++++++++++++++++++++++++++++++++++

为了适应反映表元数据后才能声明映射类的用例，提供了一个名为   :class:`.DeferredReflection`  混合的简单扩展，它改变了声明式映射过程，以便在特殊的类级别  :meth:` .DeferredReflection.prepare`  方法被调用后延迟映射过程，这将从目标数据库针对一个目标表执行反射过程，并将结果集成到声明式表映射过程中，也就是使用 ``__tablename__`` 属性的类：

    from sqlalchemy.ext.declarative import DeferredReflection
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class Reflected(DeferredReflection):
        __abstract__ = True


    class Foo(Reflected, Base):
        __tablename__ = "foo"
        bars = relationship("Bar")


    class Bar(Reflected, Base):
        __tablename__ = "bar"


.. _mapper_reflection:

基于反射的映射
-----------------------

有时，我们需要将已存在的表或者数据库映射到 SQLAlchemy 的 ORM 中。这种情况下，你可以使用 SQLAlchemy ORM 的反射机制，该机制允许自动检测表或整个数据库。对于基于反射的表映射，必须通过某种方式将列和整个表映射到 Python 类上。
两种主要实现表级的反射已经在 SQLAlchemy ORM 中得到支持—通过显式预定义类声明和推迟映射，以及零配置的基于自动映射的实现。
 
声明的类反射
==================

以 Foo 类为例，举例说明基于类声明的表反射如何工作:

.. sourcecode:: python+sql

    class Foo(Base):
        __tablename__ = 'foo'

        id = Column(Integer, primary_key=True)
        bar_id = Column(Integer, ForeignKey('bar.id'))
        value = Column(Text)

        bars = relationship('Bar', backref='foos')

    class Bar(Base):
        __tablename__ = 'bar'

        id = Column(Integer, primary_key=True)
        value = Column(Text)

execute()　方法只是执行了 SQL 语句 'CREATE TABLE foo' 。没有生成描述 foo 表列的 Python 对应类。通过创建一个混合类，你可以创建一个与自己的定义组合的类，并直接映射列定义到你自己的类定义中：

.. sourcecode:: python+sql

    class Reflected:
        """当调用 Reflected.prepare() 方法时，它就会将基类反射成关联的表并且建立映射"""

        __table_args__ = {'autoload': True}

        @classmethod
        def prepare(cls, engine):
            """将类上的映射关系关联到表以进行映射"""
            cls.metadata.bind = engine
            cls.metadata.reflect(engine)

    class Foo(Reflected, Base):
        __tablename__ = 'foo'

        bars = relationship('Bar', backref='foos')

    class Bar(Reflected, Base):
        __tablename__ = 'bar'

Reflected 类只是一个包含几个在它派生类中自动执行反射的 SQLALchemy 基类。最重要的是 __table_args__ = {'autoload': True} 选项，当该选项设置为 True 时，SQLAlchemy 就会自动反射表并将表的列映射到 Python 中的列属性中。
在 Foo 和 Bar 类中，我们不需要在映射关系中指定实际的列属性，因为 Reflected 类在 prepare() 方法中将表关联到该类，并自动将列属性映射到基类中：

.. sourcecode:: python+sql

    engine = create_engine('postgresql+psycopg2://user:pass@hostname/my_existing_database')
    Reflected.prepare(engine)

上面，我们创建了一个混合类 Reflected，它将用作我们的声明式层次结构中映射的基础。该映射不完整，直到我们这样做，即给出一个   :class:`_engine.Engine` ：

Reflected 类的目的在于定义应反映哪些类的范围。插件将搜索目标子类树以反射声明的类命名的所有表。

反射表中与映射无关并且与目标表没有任何关系的表将不会反射。

使用 Automap 反射
-----------------------

自动建立与现有的数据库表的映射，除了手工映射的方法外，还可以使用   :ref:`automap_toplevel `  扩展。这个扩展将从数据库模式中生成完整的映射类，包括基于观察到的外键约束之间的类之间的关系。虽然它包括自定义类命名和关系命名方案的钩子，但 automap 取向于零配置的方便使用。如果一个应用希望拥有一个明确的模型并使用表反射，那么 DeferredReflection 类可能更适合，因为其 less automated 的方法。

.. seealso::

      :ref:`automap_toplevel` 


.. _mapper_automated_reflection_schemes:

从反射表中自动化列命名方案
------------------------------------------

当使用任何前面描述的反射技术时，我们可以改变映射过程中的列映射命名的方式。 默认情况下，采用的是与数据库表中列的物理结构一致的使命名方式。对于不符合你的指定的规则的结果，需要自定义名字策略，例如使用下划线或小写字母。在我们使用表反射时，可以使用元编程技术，以便在收到的   :class:`_schema.Column`  参数中拦截名称，包括 ` `.key`` 属性，但也包括数据类型，如下所示：

通过使用  :meth:`_events.DDLEvents.column_reflect`  事件来将类中接收的参数与 column_info 变量关联起来，可以定义   :class:` _schema.Column` 。这个事件是最容易与正在使用的   :class:`_schema.MetaData`  对象相关联的::


    from sqlalchemy import event
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    @event.listens_for(Base.metadata, "column_reflect")
    def column_reflect(inspector, table, column_info):
        # set column.key = "attr_<lower_case_name>"
        column_info["key"] = "attr_%s" % column_info["name"].lower()

如上所述，   :class:`_schema.Column`  对象的反射将被被拦截，添加新的 ".key" 元素，比如下面的映射::

    class MyClass(Base):
        __table__ = Table("some_table", Base.metadata, autoload_with=some_engine)

这个方法同时适用于 DeferredReflection 基类以及 automap_toplevel 扩展。特别是，参见   :ref:`automap_intercepting_columns`  部分了解更多背景信息。

.. seealso::

      :ref:`orm_declarative_reflected` 

     :meth:`_events.DDLEvents.column_reflect` 

      :ref:`automap_intercepting_columns`  - 在   :ref:` automap_toplevel`  文档中


.. _mapper_primary_key:

映射到一组明确设置的主键列
-------------------------------------------

  :class:`.Mapper`  构造用于成功将表映射必须始终标识至少一个列作为该可选择性的“主键”。这样，在加载或持久化 ORM 对象时，它才能以适当的 ID 键放置在  :term:` identity map`  中。

在需要映射的反射表不包含主键约束的情况下（这种情况可能在反射场景中发生），以及一般情况下可能不存在主键列的   :ref:`针对任意可选择性的映射 <orm_mapping_arbitrary_subqueries>` ，提供了  :paramref:` .Mapper.primary_key`  参数，以便任何一组列都可以配置为该表的“主键”，就 ORM 映射而言。

以没有明确主键约束的反射表用作插图，可以通过下面的方式进行映射：

.. sourcecode:: python+sql

    from sqlalchemy import Column
    from sqlalchemy import MetaData
    from sqlalchemy import String
    from sqlalchemy import Table
    from sqlalchemy import UniqueConstraint
    from sqlalchemy.orm import DeclarativeBase


    metadata = MetaData()
    group_users = Table(
        "group_users",
        metadata,
        Column("user_id", String(40), nullable=False),
        Column("group_id", String(40), nullable=False),
        UniqueConstraint("user_id", "group_id"),
    )


    class Base(DeclarativeBase):
        pass


    class GroupUsers(Base):
        __table__ = group_users
        __mapper_args__ = {"primary_key": [group_users.c.user_id, group_users.c.group_id]}

上面代码的 group_users 表是某种类型的关联表，具有字符串列 user_id 和 group_id，但没有设置主键；相反，只有一个   :class:`.UniqueConstraint`  约束，确定这两个列表示一个唯一的键。类似地, 我们不必在映射关系中指定实际的主键。

.. _include_exclude_cols:

映射表的一个子集的列
----------------------------------------------

有时反射表可能包含不会在需要的领域上使用的列，可安全忽略。对于这样的表，其具体列不会被 ORM 引用。功能  :paramref:`_orm.Mapper.include_properties`  或  :paramref:` _orm.Mapper.exclude_properties`  参数可指示只在映射期间包括每个列的子集，其中来自目标   :class:`_schema.Table`  的其他列将不被 ORM 以任何方式考虑。示例::

    class User(Base):
        __table__ = user_table
        __mapper_args__ = {"include_properties": ["user_id", "user_name"]}

在上面的示例中，该 User 类将映射到 user_table 表，并只包括 user_id 和 user_name 两列 - 其他列不会被引用。

同样的方式：

    class Address(Base):
        __table__ = address_table
        __mapper_args__ = {"exclude_properties": ["street", "city", "state", "zip"]}

将把 Address 类映射到 address_table 表，并包括除 street、city、state 和 zip 以外的所有列。

如示例中所示，列的引用方式可以是字符串名称或直接引用   :class:`_schema.Column`  对象。直接引用可能对于明确性或解析映射到可能具有重复名称的多表结构时有用。::

    class User(Base):
        __table__ = user_table
        __mapper_args__ = {
            "include_properties": [user_table.c.user_id, user_table.c.user_name]
        }

当列未包含在映射中时，这些列将不会在任何 SELECT 语句中引用，即使你执行   :func:`_sql.select`  或旧的   :class:` _query.Query`  对象，也不会有映射属性在映射的类中来表示该列。为该名称分配属性将没有影响，仅有记忆作用。

但是，重要的事情要注意：**schema level column defaults 仍然适用于这些包括它们的   :class:`_schema.Column`  对象，即使他们被从 ORM 映射中排除**。

"schema level column defaults" 指的是在   :ref:`metadata_defaults`  中描述的默认值，包括  :paramref:` _schema.Column.default` 、  :paramref:`_schema.Column.onupdate`  、  :paramref:` _schema.Column.server_default`   和  :paramref:`_schema.Column.server_onupdate`  参数。这样构造物继续正常工作，因为在  :paramref:` _schema.Column.default`  和  :paramref:`_schema.Column.onupdate`  的情况下，  :class:` _schema.Column`  对象仍然存在于底层的   :class:`_schema.Table`  上，因此允许在 ORM 发出 INSERT 或 UPDATE 时使用默认函数，在  :paramref:` _schema.Column.server_default`  和  :paramref:`_schema.Column.server_onupdate`  的情况下，关系数据库本身会发出这些默认值，作为服务器端行为。
