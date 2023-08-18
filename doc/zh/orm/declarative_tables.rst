.. _orm_declarative_table_config_toplevel:

=============================================
使用 Declarative 进行表配置
=============================================

如 :ref:`orm_declarative_mapping` 中所述，Declarative 样式提供了一个能够同时生成映射 :class:`_schema.Table` 对象的方法，也可以直接容纳 :class:`_schema.Table` 或其他的 :class:`_sql.FromClause` 对象。

以下示例假定一个使用以下方式定义 Declarative 基类的 style：

    from sqlalchemy.orm import DeclarativeBase

    class Base(DeclarativeBase):
        pass

所有以下示例中的示例都展示了从上述 Base 继承的类。:: 

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

在上述示例中，Declarative 将构建一个等效于以下内容的 :class:`_schema.Table` 对象::

    # 相应的 Table 对象
    user_table = Table(
        "user",
        Base.metadata,
        Column("id", Integer, primary_key=True),
        Column("name", String(50)),
        Column("fullname", String()),
        Column("nickname", String(30)),
    )

在上面的示例中，当映射 "User" 类时，可以通过 ``__table__`` 属性直接访问这个 :class:`_schema.Table` 对象；这在 :ref:`orm_declarative_metadata` 中有进一步的说明。

.. sidebar::  ``mapped_column()`` 将 Column 废弃

    在 SQLAlchemy 1.x 的用户会注意到新的 :func:`_orm.mapped_column` 构造, 这是从 SQLAlchemy 2.0 系列开始的新映射。这个 ORM 特定的构造是为 Declarative 映射而设计的，首先和最重要的是除了在 Declarative 映射中使用 :class:`_schema.Column` 的功能外，还添加了 ORM 特定的便利特性，例如在构造中建立 :paramref:`_orm.mapped_column.deferred` 的能力，并且最重要的是提示像 Mypy_ 和 Pylance_ 这样的类型检查工具，在类级别和实例级别忠实地表示属性的运行时行为。正如后面的章节将会看到的，它也是 SQLAlchemy 2.0 中新的基于注释的配置风格的前景。

    旧代码的用户应该知道，在 Declarative 中 :class:`_schema.Column` 的形式始终以它一直使用的方式进行工作。不同的属性映射形式也可以在单个映射中根据属性逐个混合，因此可以以任何步伐进行迁移到新形式。有关将 Declarative 模型迁移到新形式的详细步骤，请参阅 :ref:`whatsnew_20_orm_declarative_typing` 章节。

:func:`_orm.mapped_column` 构造接受 :class:`_schema.Column` 构造接受的所有参数，以及其他特定于 ORM 的参数。 :paramref:`_orm.mapped_column.__name` 字段通常被省略，该字段表示数据库列的名称，因为在 Declarative 过程中，使用给定到构造的属性名称并将其分配为列的名称（在上面的示例中，这指的是名称 "id"、名称 "name"、名称 "fullname" 和名称 "nickname"）。分配强制 :paramref:`_orm.mapped_column.__name` 的备用名称也是有效的，其中生成的 :class:`_schema.Column` 将在 SQL 和 DDL 语句中使用给定名称，在这种情况下，映射类将继续使用属性名称，独立于列本身的名称。更多信息请参见 :ref:`mapper_column_distinct_names`。

.. tip::

    :func:`_orm.mapped_column` 构造 **仅在 Declarative 类映射中有效**。在使用核心进行构造 :class:`_schema.Table` 对象时，以及在使用 :ref:`imperative table <orm_imperative_table_configuration>` 配置时，仍需要 :class:`_schema.Column` 构造函数以指示具有数据库列的存在。

.. seealso::

    :ref:`mapping_columns_toplevel` - 包含有关影响如何解释传入的 :class:`.Column` 对象的其他说明。

.. _orm_declarative_mapped_column:

使用 Annotated Declarative Table（对 "mapped_column()" 进行类型注释的类型注释格式）
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:func:`_orm.mapped_column` 构造能够从关联的在 Declarative 映射类中声明的 attribute 上使用 `PEP 484`_ 类型注释派生出其列配置信息。如果使用，必须在称为 :class:`_orm.Mapped` 的特殊 SQLAlchemy 类型中存在这些类型注释。

以下展示了前面一节中的映射，增加了对 :class:`_orm.Mapped` 的使用 ::


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

在上述示例中，Declarative 处理每个类 attribute 时，每个 :class:`_orm.mapped_column` 将从相应的 :class:`_orm.Mapped` 类型注释中的左侧派生出额外的参数。此外，当遇到一个没有分配给属性的 :class:`_orm.Mapped` 类型注释时（这种形式受到类似于 Python dataclasses 的类似样式的启发），Declarative 将隐式地生成一个空的 :func:`_orm.mapped_column` 指令，该指令然后从存在的 :class:`_orm.Mapped` 注释派生其配置。

在上面的示例中，当声明这个类时，Declarative 映射处理过程将生成一个新的 :class:`_schema.Table` 对象，并自动与与 Declarative ``Base`` 相关联的 :class:`_schema.MetaData` 集合一起创建。接着，每个 :func:`_orm.mapped_column` 实例都将用于在此过程期间生成 :class:`_schema.Column` 对象，这将成为此 :class:`_schema.Table` 对象的 :attr:`.schema.Table.columns` 集合的一部分。

.. _orm_declarative_mapped_column_nullability:

``mapped_column()`` 会将数据类型和可为 Null 的行从 ``Mapped`` 注释中派生出来
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

：func:`_orm.mapped_column` 从 :class:`_orm.Mapped` 注释中提取的两个属性是：

* **数据类型**-:class:`_orm.mapped_column` 从 :class:`_orm.Mapped` 中派生的 Python 类型，作为包含在其中的 `typing.Optional` 构造（如果存在）中的特定 :class:`_sqltypes.TypeEngine` 子类（例如， :class:`.Integer`、:class:`.String`、 :class: `.DateTime` 或 :class:`.Uuid`) 关联。

    数据类型是根据 Python 类型到 SQLAlchemy 数据类型的字典来确定的。可以完全自定义此字典，正如下一节 :ref:`orm_declarative_mapped_column_type_map` 中所详细说明的那样。默认类型映射是由下面的代码示例实现的：

      from typing import Any
      from typing import Dict
      from typing import Type

      import datetime
      import decimal
      import uuid

      from sqlalchemy import types

      # default type mapping, deriving the type for mapped_column()
      # from a Mapped[] annotation
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

    如果 :func:`_orm.mapped_column` 构造显式地指示封送到 :paramref:`_orm.mapped_column.__type` 参数的类型，则会忽略给定的 Python 类型。

* **可为 Null** - 当 :paramref:`_orm.mapped_column.nullable` 参数出现时，将使用 :func:`_orm.mapped_column` 指示为 ``NULL`` 或 ``NOT NULL``。此外，如果 :paramref:`_orm.mapped_column.primary_key` 参数出现并设置为 ``True``，则还将意味着此列应为 ``NOT NULL``。

  在没有这两个参数的情况下，如果在 :class:`_orm.Mapped` 类型注释中存在 `Optional[]`，则表明应使用 `NULL`，否则表明应使用 `NOT NULL`。如果根本不存在 `Mapped[]` 注释，并且不存在 :paramref:`_orm.mapped_column.nullable` 或 `:paramref:`_orm.mapped_column.primary_key` 参数，则使用 :class:`_schema.Column` 的 SQLAlchemy 默认值 ``NULL``。

  在下面的示例中，id 和 data 列将是 ``NOT NULL``，而 additional_info 列将是 ``NULL``::

      from typing import Optional

      from sqlalchemy.orm import DeclarativeBase
      from sqlalchemy.orm import Mapped
      from sqlalchemy.orm import mapped_column

      class Base(DeclarativeBase):
          pass

      class SomeClass(Base):
          __tablename__ = "some_table"

          # primary_key=True, 因此将是 NOT NULL
          id: Mapped[int] = mapped_column(primary_key=True)

          # 不是 Optional[]，因此将是 NOT NULL
          data: Mapped[str]

          # Optional[]，因此将是 NULL
          additional_info: Mapped[Optional[str]]

对于将硬编码字典作为 :meth:`.TypeEngine.with_variant` 中的值只有一种配置。下一节将描述第二种方法。

.. _orm_declarative_mapped_column_type_map:

自定义类型映射
~~~~~~~~~~~~~~~~~~~~~~~~
Python 类型到 SQLAlchemy :class:`_types.TypeEngine` 类型的映射，已在上一节中描述，默认值为 "sqlalchemy.sql.sqltypes" 模块中的硬编码字典。然而，当协调 Declarative 映射过程的 :class:`_orm.registry` 对象在构造 :class:`_orm.registry` 对象时先查看了本地的用户定义的类型字典，可通过传递 :class:`_orm.registry.type_annotation_map` 参数来与 :class:`_orm.DeclarativeBase` 超类相关联时，这个字典可能与之相关联。

例如，如果我们希望将 Python ``int`` 的默认 :class:`.Integer` 映射到 ``BigInt``，使用如下的方式将 :class:`_sqltypes.TIMESTAMP` 映射到具有 ``timezone=True`` 的 ``TIMESTAMP``， 并且只需要在 Microsoft SQL Server 上使用 Python 的 ``str`` 时使用 :class:`_sqltypes.NVARCHAR` 映射，代码可以这样写：

    import datetime
    from decimal import Decimal
    from typing import Any
    from typing import Dict
    from typing import Type
    import uuid
    from sqlalchemy import types

    # default type mapping, deriving the type for mapped_column()
    # from a Mapped[] annotation
    type_map: Dict[Type[Any], TypeEngine[Any]] = {
        bool: types.Boolean(),
        bytes: types.LargeBinary(),
        datetime.date: types.Date(),
        datetime.datetime: types.DateTime(),
        datetime.time: types.Time(),
        datetime.timedelta: types.Interval(),
        decimal.Decimal: types.Numeric(),
        float: types.Float(),
        int: types.BigInteger(), # Using BigInteger Here
        str: types.String(),
        uuid.UUID: types.Uuid(),
    }
    type_map["mssql"] = {
        str: types.String().with_variant(types.NVARCHAR, "mssql"),
        datetime.datetime: types.TIMESTAMP(timezone=True),
    }

然后，我们就可以这样进行数据库表的创建：

    from sqlalchemy.schema import CreateTable
    from sqlalchemy.dialects import mssql, postgresql
    smt = CreateTable(User.__table__)
    print(smt.compile(dialect=mssql.dialect()))



动态映射多个类型配置
~~~~~~~~~~~~~~~~~~~~~~~~~~~

如上所述，可以使用 :paramref:`_orm.registry.type_annotation_map` 字典将单个 Python 类型与任何类型的 SQLAlchemy :class：`_types.TypeEngine` 配置相关联。另外一种做法是使用 Python 的 typing 系统，通过使用 :pep:`593` ``Annotated`` 通用类型，将附加的元数据捆绑在一起。这使我们能够将单个 Python 类型与基于附加类型限定符的多个 SQL 类型的不同变体进行组合。一个典型的例子是将 Python ``str`` 数据类型映射到具有不同长度的 ``VARCHAR`` SQL 数据类型上。另一个例子是将不同版本的 ``decimal.Decimal`` 映射到不同大小的 ``NUMERIC`` 列上。

如下代码中使用到的 :meth:`.TypeEngine.with_variant` 方法可以将一系列在多个列上使用的参数缩减为最短的形式。我们可以将这些配置组成 :func:`_orm.mapped_column` 实例，然后将其直接打包到 ``Annotated`` 实例中，然后在多个类定义中重复使用，Declarative 将在提供此类时解压缩 ``Annotated`` 对象，跳过任何与 SQLAlchemy 不相关的指令，仅搜索 SQLAlchemy ORM 构造。

以下示例展示了在这种方式下使用的各种预配置字段类型，在这里, 我们定义了 "intpk"，它代表一个具有 :class:`.Integer` 数据类型的主键，"timestamp"，它表示 :class:`.DateTime` 类型，该类型将使用 ``CURRENT_TIMESTAMP`` 作为 DDL 级别的列默认值，和 "required_name"，它是一个长度为 30 的 :class:`.String`，它是 ``NOT NULL``::

    from datetime import datetime
    from typing_extensions import Annotated
    from sqlalchemy import func
    from sqlalchemy import String
    from sqlalchemy.orm import mapped_column

    intpk = Annotated[int, mapped_column(primary_key=True)]
    timestamp = Annotated[
        datetime,
        mapped_column(nullable=False, server_default=func.CURRENT_TIMESTAMP()),
    ]
    required_name = Annotated[str, mapped_column(String(30), nullable=False)]

上面的 ``Annotated`` 对象可以直接在 :class:`_orm.Mapped` 中使用，其中预配置的 :func:`_orm.mapped_column` 构造将被提取并复制到每个属性特定的新实现中：

    class SomeClass(Base):
        __tablename__ = "some_table"

        short_name: Mapped[str_30] = mapped_column(primary_key=True)
        long_name: Mapped[str_50]
        num_value: Mapped[num_12_4]
        short_num_value: Mapped[num_6_2]

在使用 ``Annotated`` 类型进行此种方式使用时，类型的配置也可以受到每个属性的影响。对于上述类型使用了 :paramref:`_orm.mapped_column.nullable` 的情况，我们可以向这些类型中的任意一个应用 ``Optional[]`` 通用修饰符，这样该字段即可在 Python 级别可选或不可选，这将独立于数据库中的 ``NULL`` / ``NOT NULL`` 设置。例如：

    from typing_extensions import Annotated
    from datetime import datetime
    from typing import Optional

    from sqlalchemy.orm import DeclarativeBase

    timestamp = Annotated[datetime, mapped_column(nullable=False)]

    class Base(DeclarativeBase):
        pass

    class SomeClass(Base):
        __tablename__ = "some_table"

        # 在 pep-484 类型上会是 Optional，但是在数据库中是 NOT NULL
        created_at: Mapped[Optional[timestamp]]

类似地，还可以有 :paramref:`_orm.mapped_column.nullable` 参数与与之相反的可选 / 不可选属性的 :func:`_orm.mapped_column`，例如：将写入数据库列但值为 ``None`` 的 ORM 映射属性注释为在 Python 级别为允许可选属性的情况。 :

    class SomeClass(Base):
        # ...

        # 将是 String() NOT NULL，但是在 Python 中可以是 None
        data: Mapped[Optional[str]] = mapped_column(nullable=False)

类似地，一个非 None 的属性可以写入数据库列，但是却需要以某些原因为 Null，可以将 :paramref:`_orm.mapped_column.nullable` 参数设置为 ``True``：

    class SomeClass(Base):
        # ...

        # 在类型检查器中，属性将不会期望成为 None，
        # 但是，将作为 NULL 的 String()
        data: Mapped[str] = mapped_column(nullable=True)


.. _orm_declarative_mapped_column_enums:

在类型映射中使用 Python ``Enum`` 或 pep-586 ``Literal`` 类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. versionadded:: 2.0.0b4 - 添加了 ``Enum`` 支持

.. versionadded:: 2.0.1 - 添加了 ``Literal`` 支持

用户定义的 Python 类型从 Python 内置的 ``enum.Enum`` 或 ``typing.Literal`` 类派生时，当在 ORM 声明性映射中使用时，它们会自动链接到 SQLAlchemy :class:`.Enum` 数据类型。下面的示例使用自定义 ``enum.Enum``：

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

在上述示例中，映射属性 ``SomeClass.status`` 将链接到具有数据类型 ``Enum(Status)`` 的 :class:`.Column`。例如，我们可以在 PostgreSQL 数据库中的 CREATE TABLE 输出中查看此列所做的映射：

.. sourcecode:: sql

  CREATE TYPE status AS ENUM ('PENDING', 'RECEIVED', 'COMPLETED')

  CREATE TABLE some_table (
    id SERIAL NOT NULL,
    status status NOT NULL,
    PRIMARY KEY (id)
  )

类似地，可以使用 :class:`typing.Literal`，使用 '' 中的所有字符串：

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

在 :paramref:`_orm.registry.type_annotation_map` 中使用的条目链接到基础类型。``enum.Enum`` Python类型和``typing.Literal``类型可以使用一种特殊形式，将其映射为SQLAlchemy的:class:`.Enum` SQL类型，这个特殊形式可以告诉:class:`.Enum`数据类型自动针对任意枚举类型进行配置。默认情况下，这个隐式的配置应该以下面的形式指示出来：

    import enum
    import typing

    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        type_annotation_map = {
            enum.Enum: sqlalchemy.Enum(enum.Enum),
            typing.Literal: sqlalchemy.Enum(enum.Enum),
        }

Declarative内的解析逻辑可以将``enum.Enum``的子类以及``typing.Literal``的实例解析为与``enum.Enum``或``typing.Literal``在:paramref:`_orm.registry.type_annotation_map`字典中匹配。 :class:`.Enum` SQL类型然后知道如何生成一个已配置的版本，其中包括默认字符串长度。如果传递了一个由非字符串值组成的``typing.Literal``，则会引发错误。

本地枚举和命名
+++++++++++++++++++

:paramref:`.sqltypes.Enum.native_enum`参数指的是:class:`.sqltypes.Enum`数据类型是否应创建所谓的“本地”枚举，在MySQL/MariaDB上是“ENUM”数据类型，在PostgreSQL上则是由“CREATE TYPE”创建的新“TYPE”对象，或者是“非本地”枚举，这意味着将使用``VARCHAR``来创建数据类型。对于MySQL/MariaDB或PostgreSQL以外的后端，在所有情况下使用``VARCHAR``（第三方方言可能具有自己的行为）。

因为PostgreSQL的``CREATE TYPE``要求存在类型的显式名称，所以在不指定显式:class:`_sqltypes.Enum`数据类型在映射中对于隐式生成的:class:`.sqltypes.Enum`，特殊的回退逻辑存在：

1. 如果:class:`.sqltypes.Enum`与``enum.Enum``对象相关联，则:paramref:`.Enum.native_enum`参数默认为``True``，并且枚举的名称将取自``enum.Enum``数据类型的名称。PostgreSQL后端将使用该名称假定``CREATE TYPE``。
2. 如果:class:`.sqltypes.Enum`与``typing.Literal``对象相关联，则:paramref:`.Enum.native_enum`参数默认为``False``。不会生成名称，假定使用``VARCHAR``。

要将带有PostgreSQL ``CREATE TYPE``类型的``typing.Literal``，必须使用显式:class:`.sqltypes.Enum`，可以在类型映射中使用：

    import enum
    import typing

    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase

    Status = Literal["pending", "received", "completed"]


    class Base(DeclarativeBase):
        type_annotation_map = {
            Status: sqlalchemy.Enum("pending", "received", "completed", name="status_enum"),
        }

或者在下面的:func:`_orm.mapped_column`中：

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

修改默认枚举类型的配置
+++++++++++++++++++++++++++++++++++++++++++++++

为了修改隐式生成的:class:`.enum.Enum`数据类型的固定配置，指定在:paramref:`_orm.registry.type_annotation_map`中添加条目即可，表明存在附加参数如何表示。例如，要无条件地使用“非本地枚举”，可以将:paramref:`.Enum.native_enum`参数设置为False，以应用于所有类型：

    import enum
    import typing
    import sqlalchemy
    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        type_annotation_map = {
            enum.Enum: sqlalchemy.Enum(enum.Enum, native_enum=False),
            typing.Literal: sqlalchemy.Enum(enum.Enum, native_enum=False),
        }

.. versionchanged:: 2.0.1  实现了支持在建立:paramref:`_orm.registry.type_annotation_map`时重写参数（如:paramref:`_sqltypes.Enum.native_enum`）的功能。以前，此功能无法正常工作。

要为特定的``enum.Enum``子类型使用特定配置，例如在使用示例``Status``数据类型时将字符串长度设置为50：

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

将特定的``enum.Enum``或`` typing.Literal``链接到其他数据类型
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

上面的示例展示了如何使用:class:`_sqltypes.Enum`来自动配置自身来生成 :class:`.Enum` SQL 类型的映射，用于``enum.Enum``或``typing.Literal``类型的所有变体。对于特定的``enum.Enum``或``typing.Literal``应链接到其他类型的使用案例中，这些特定类型也可以放置在类型映射中。在下面的示例中，在没有仅包含字符串值的``Literal``变体的情况下，将``Literal[]``输入与:class:`_sqltypes.JSON`数据类型类对应：

    from typing import Literal

    from sqlalchemy import JSON
    from sqlalchemy.orm import DeclarativeBase

    my_literal = Literal[0, 1, True, False, "true", "false"]


    class Base(DeclarativeBase):
        type_annotation_map = {my_literal: JSON}

在上述配置中，``my_literal``数据类型将解析为:class:`._sqltypes.JSON`实例。其他``Literal``变体将继续解析为:class:`_sqltypes.Enum`数据类型。


``mapped_column()``中的dataclass功能
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:func:`_orm.mapped_column`构造函数与SQLAlchemy的“原生Python数据类”功能集成，这个功能集成在
:ref:`orm_declarative_native_dataclasses`中讨论。有关附加指令支持的当前信息请参见该部分。


.. _orm_declarative_metadata:

访问表和元数据
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

映射到声明性类将始终包含称为``__table__``的属性；当使用上面的``__tablename__``配置完成时，声明性过程会通过``__table__``属性将:class:`_schema.Table`提供出来：

    # 访问表
    user_table = User.__table__

上面的表最终与 :attr:`_orm.Mapper.local_table`属性对应，我们可以通过运行时检查系统 :ref:`inspection_toplevel` 中查看：

    from sqlalchemy import inspect

    user_table = inspect(User).local_table

:class:`_schema.MetaData`集合与声明式的:class:`_orm.registry`以及基类一起使用，通常需要运行DDL操作，例如创建表格，以及与诸如Alembic之类的迁移工具一起使用。该对象可以通过声明式基类的``.metadata``属性以及:class:`_schema.MetaData`集合进行获取。下面是一个小脚本示例，我们希望针对SQLite数据库发布所有表格的CREATE：

    engine = create_engine("sqlite://")

    Base.metadata.create_all(engine)

.. _orm_declarative_table_configuration:

声明式表配置
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在Declarative表配置中，应该使用``__table_args__``声明性类属性提供要传递到:class:`_schema.Table`构造函数的其他参数。此属性可以采用以下两种形式之一。一个是作为字典：

    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = {"mysql_engine": "InnoDB"}

另一个是作为一个元组，其中每个参数都是位置参数（通常是约束）：

    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = (
            ForeignKeyConstraint(["id"], ["remote_table.id"]),
            UniqueConstraint("foo"),
        )

关键字参数可以通过使用指定 ``__table_args__`` 声明性类属性的最后一个参数作为字典来进行指定：

    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = (
            ForeignKeyConstraint(["id"], ["remote_table.id"]),
            UniqueConstraint("foo"),
            {"autoload": True},
        )

类还可以使用:func:`_orm.declared_attr`方法装饰符以动态方式指定``__table_args__``声明性属性以及``__tablename__``属性。参见:ref:`orm_mixins_toplevel`获取背景。

.. _orm_declarative_table_schema_name:

在声明性表中明确模式名称
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在 :ref:`schema_table_schema_name`文档中指出，:class:`_schema.Table`的模式名称适用于一个单独的:class:`_schema.Table` 对象,使用 :paramref:`_schema.Table.schema` 参数。当使用Declarative表时，此选项如其他参数一样传递给 ``__table_args__``字典：

    from sqlalchemy.orm import DeclarativeBase


    class Base(DeclarativeBase):
        pass


    class MyClass(Base):
        __tablename__ = "sometable"
        __table_args__ = {"schema": "some_schema"}

在所有情况下，会将枚举的元类型显示为名称。例如，

    from enum import Enum

    class Color(str, Enum):
        RED = "red"
        GREEN = "green"
        BLUE = "blue"

这个枚举会定义三个常量：``Color.RED``，``Color.GREEN``和``Color.BLUE``。每个常量是一个:class:`enum.Enum`实例，它继承了元类型:class:`enum.Enum`并定义了一个表示枚举成员的字符串值。

在SQLAlchemy 1.4之前，:class:`enum.Enum`被映射到数据库中的SQLAlchemy :class:`.Enum`类型。默认情况下，当映射到数据库时，类型名被设置为元类型的名称。例如，在幕后，以下模式会被生成：``CREATE TYPE color AS ENUM ('red', 'green', 'blue');``, 其中``color``是类型的名称，而字符串值是它的成员。

''':class:`typing.Literal`'' 实例不能像其他 :class:`enum.Enum` 实例一样被映射到数据库中的 :class:`.Enum` 类型。取而代之的是，对于一个类似``Literal["spam", "ham"]``的字面值类型，表示枚举成员的字符串将直接写入Schemae。因此，以下幕后SQL将被生成：``CREATE TYPE <name> AS ENUM ('spam', 'ham');``。与前面枚举类型的情况相似，这个``<name>``仅用于PostgreSQL和MariaDB/MySQL，其他的情况下，这个类型只是一个varchar并没有类型。在上述代码中，我们创建了一个 mixin 类 "Reflected"，该类将作为我们的声明式层次结构中基类，当 "Reflected.prepare" 方法被调用时，这些类将变成映射。在给定一个 :class:`_engine.Engine` 的情况下，上述映射是不完整的::

    engine = create_engine("postgresql+psycopg2://user:pass@hostname/my_existing_database")
    Reflected.prepare(engine)

"Reflected" 类的目的是定义映射应该被反射的范围。该插件将搜索目标子类树以及声明类命名的所有表。那些不是映射的表，或者与目标表通过外键约束关联的表，在目标数据库中将不会被反映。

使用 Automap
^^^^^^^^^^^^^^

映射到使用表反射的现有数据库的更自动化的解决方案是使用 :ref:`automap_toplevel` 扩展程序。此扩展程序将从数据库模式生成整个映射类，并根据观察到的外键约束生成类之间的关系。虽然它包括自定义的挂钩（例如允许自定义类命名和关系命名方案的挂钩），但 automap 是面向迅速零配置的工作风格。如果应用程序希望具有完全显式的模型，其中使用表反射，那么 :ref:`DeferredReflection <orm_declarative_reflected_deferred_reflection>` 类可能更适合，因为它采用的是更少的自动化方法。

.. seealso::

    :ref:`automap_toplevel`


.. _mapper_automated_reflection_schemes:

从反射表自动命名列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当使用任何先前的反射技术时，我们都可以改变列映射的命名方案。:class:`_schema.Column` 对象包括一个参数 :paramref:`_schema.Column.key`，它是一个字符串名称，独立于列的 SQL 名称，用于确定该:class:`_schema.Column` 将在表的 :attr:`_schema.Table.c` 集合中以什么名称存在。如果不提供该键名，则 :class:`_orm.Mapper` 将使用该键作为 :class:`_schema.Column` 映射的属性名。当使用表反射时，我们可以使用 :meth:`_events.DDLEvents.column_reflect` 事件在接收到 :class:`_schema.Column` 的参数时进行拦截，并应用我们需要进行的任何更改，包括“可”属性，但也包括数据类型。

该事件钩子最容易与 :class:`_schema.MetaData` 对象相关联，如下例所示::

    from sqlalchemy import event
    from sqlalchemy.orm import DeclarativeBase

    class Base(DeclarativeBase):
        pass

    @event.listens_for(Base.metadata, "column_reflect")
    def column_reflect(inspector, table, column_info):
        # 设置column.key="attr_<lower_case_name>"
        column_info["key"] = "attr_%s" % column_info["name"].lower()

使用以上事件，将被反射的 :class:`_schema.Column` 对象将被我们的事件拦截，从而添加一个新的".key"元素，例如以下映射::

    class MyClass(Base):
        __table__ = Table("some_table", Base.metadata, autoload_with=some_engine)

该方法也适用于 :class:`.DeferredReflection` 基础类，以及 :ref:`automap_toplevel` 扩展程序。对于 automap，特别是参见 :ref:`automap_intercepting_columns` 部分。

.. seealso::

    :ref:`orm_declarative_reflected`

    :meth:`_events.DDLEvents.column_reflect`

    :ref:`automap_intercepting_columns` - 在 :ref:`automap_toplevel` 文档中


.. _mapper_primary_key:

映射到一组明确的主键列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

:class:`.Mapper` 构造在成功映射表格时总是需要至少一个列被标识为该可选择的唯一 "主键"。这是为了在加载或持久化 ORM 对象时，可以使用适当的 "identity key" 将其放置在 :term:`identity map` 中。

在反映表（reflected table）没有设置主键约束的情况下（在反射场景中可能会出现），以及在:ref:`mapping against arbitrary selectables <orm_mapping_arbitrary_subqueries>`的一般情况中，:paramref:`.Mapper.primary_key` 参数提供了任何一组列都可以配置为表格的“主键”。作为 ORM 映射关系。

例如，对于使用现有 :class:`.Table` 对象进行的 Imperative Table 映射，当表格没有设置任何声明的主键（可能会发生在反射场景中），我们可以按以下方式映射此类表格::

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

在上述示例中，" group_users" 表是某种将字符串列 "user_id" 和 "group_id" 连接起来的协会表格，但是没有设置主键；相反，只有一个 :class:`.UniqueConstraint` 建立了这两列表示唯一键。:class:`.Mapper` 不会自动检查唯一约束以获取主键；相反，我们使用 :paramref:`.Mapper.primary_key` 参数，传递一个 ``[group_users.c.user_id, group_users.c.group_id]`` 集合，表示这两个列应该用于构建 "GroupUsers" 类型实例的 "identity key"。

.. _include_exclude_cols:

映射表中的一个子集列
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

有时候表反射可能提供了一个有一些并不重要且可以安全忽略的列的 :class:`_schema.Table`，:paramref:`_orm.Mapper.include_properties` 或 :paramref:`_orm.Mapper.exclude_properties` 参数可以指示只映射子集的列，目标 :class:`_schema.Table` 中的其他列将不会被 ORM 考虑。例如:: 

    class User(Base):
        __table__ = user_table
        __mapper_args__ = {"include_properties": ["user_id", "user_name"]}

在上面的示例中，“User”类将映射到 “user_table”表，只包括“user_id”和“user_name”列，其余的不引用。

同样::

    class Address(Base):
        __table__ = address_table
        __mapper_args__ = {"exclude_properties": ["street", "city", "state", "zip"]}

将映射“Address”类到“address_table”表格，包含所有存在的列，除了 "street"、"city"、"state" 和 "zip"。

如示例所示，列可以通过字符串名称或直接引用 :class:`_schema.Column` 对象来引用。直接引用列对象可能有助于明确性，也可以解决映射到可能具有重复名称的多表结构中时存在的歧义。

例如::

    class User(Base):
        __table__ = user_table
        __mapper_args__ = {
            "include_properties": [user_table.c.user_id, user_table.c.user_name]
        }

当列未包含在映射中时，这些列在执行 :func:`_sql.select` 或旧版 :class:`_query.Query` 对象时不会在任何 SELECT 语句中被引用，也不会在映射的类上存在任何映射的属性。指定该名称的属性将只是一个普通的 Python 属性分配，其效果不会超出通常的 Python 属性分配。

注意，**模式级别的列默认值仍然有效**，对于包含这些默认值的 :class:`_schema.Column` 对象尤其是对那些被排除在外的列仍然有效。

"模式级别的列默认值" 是指在 :ref:`metadata_defaults` 中描述的默认值，其中包括通过 :paramref:`_schema.Column.default`，:paramref:`_schema.Column.onupdate`、:paramref:`_schema.Column.server_default` 和 :paramref:`_schema.Column.server_onupdate` 参数进行配置的默认值。这些构造物之所以继续起作用是因为对于 :paramref:`_schema.Column.default` 和 :paramref:`_schema.Column.onupdate`，即使在 ORM 发出 INSERT 或 UPDATE 时，:class:`_schema.Column` 对象仍然存在于底层的 :class:`_schema.Table` 上，从而允许默认函数在 ORM 发出 INSERT 或 UPDATE 时进行处理，在 :paramref:`_schema.Column.server_default` 和 :paramref:`_schema.Column.server_onupdate` 的情况下，联接式数据库本身作为服务器端行为发出这些默认值。