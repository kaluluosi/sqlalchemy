使用 Mixin 构建映射层次结构
========================================

使用   :ref:`Declarative<orm_declarative_mapping>`  风格映射类时，常见的需求是共享普通功能，
比如特定的列、表或映射器选项、命名方案或其他映射属性。使用声明性映射时，可以通过使用  :term:`mixin 类`  或增强声明性基类本身的方法来支持此约定。

.. tip:: 除了 mixin 类之外，还可以使用  :pep:`593`  中的 ` `Annotated`` 类型在多个类之间共享公共列选项；请参见   :ref:`orm_declarative_mapped_column_type_map_pep593`  和   :ref:` orm_declarative_mapped_column_pep593` ，了解关于这些 SQLAlchemy 2.0 功能的背景信息。 

以下是一些常见的混用习惯的示例::

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import declared_attr
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class CommonMixin:
        """定义一系列通用元素，这些元素可以通过使用这个类作为 mixin 类，并应用到映射类中。

        """

        @declared_attr.directive
        def __tablename__(cls) -> str:
            return cls.__name__.lower()

        __table_args__ = {"mysql_engine": "InnoDB"}
        __mapper_args__ = {"eager_defaults": True}

        id: Mapped[int] = mapped_column(primary_key=True)


    class HasLogRecord:
        """标记类，该类与 LogRecord 类之间有一对多关系。"""

        log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

        @declared_attr
        def log_record(self) -> Mapped["LogRecord"]:
            return relationship("LogRecord")


    class LogRecord(CommonMixin, Base):
        log_info: Mapped[str]


    class MyModel(CommonMixin, HasLogRecord, Base):
        name: Mapped[str]

上述示例说明了一个名为 ``MyModel`` 的类，该类在其基类中包括两个 Mixin 类 ``CommonMixin`` 和 ``HasLogRecord``，以及一个补充类 ``LogRecord``，它也包括 ``CommonMixin``，演示了一系列对于 Mixin 和基类都支持的构造的构造，包括：

* 使用   :func:`_orm.mapped_column` 、  :class:` _orm.Mapped`  或   :class:`_schema.Column`  声明的列将从 mixin 或基类复制到要映射的目标类上；上述示例通过列属性 ` `CommonMixin.id`` 和 ``HasLogRecord.log_record_id`` 进行了说明。
* Declarative 指令，如 ``__table_args__`` 和 ``__mapper_args__`` 可以分配给 mixin 或基类，在继承自 mixin 或基类的任何类中都会自动生效。上述示例使用了 ``__table_args__`` 和 ``__mapper_args__`` 属性说明了这个特性。
* 所有 Declarative 指令，包括所有的 ``__tablename__``, ``__table__``, ``__table_args__`` 和 ``__mapper_args__``, 都可以使用用户定义的类方法来实现，这些方法被修饰符   :class:`_orm.declared_attr`  装饰(具体见  :attr:` _orm.declared_attr.directive`  成员)。上述例子使用了一个 ``def __tablename__(cls)`` 类方法，动态地生成一个   :class:`.Table`  名称；当应用于 ` `MyModel`` 类时，表名将生成为 ``"mymodel"``，而当应用于 ``LogRecord`` 类时，表名将生成为 ``"logrecord"``。
* 也可以使用用户定义的类方法，在将修饰符   :class:`_orm.declared_attr`  调用此类方法时，在要映射的目标类上生成其他 ORM 属性，如   :func:` _orm.relationship` 。上述例子使用一个 many-to-one   :func:`_orm.relationship`  生成一个映射对象，该映射对象称为 ` `LogRecord``。

上述特性都可以通过一个   :func:`_sql.select`  示例来展示：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> print(select(MyModel).join(MyModel.log_record))
    {printsql}SELECT mymodel.name, mymodel.id, mymodel.log_record_id
    FROM mymodel JOIN logrecord ON logrecord.id = mymodel.log_record_id

.. tip::   :class:`_orm.declared_attr`  的示例将尝试说明每个方法示例的正确  :pep:` 484`  注释。对于   :class:`_orm.declared_attr`  函数，注释使用是 **完全可选的**，并且不会被 Declarative 使用；但是，这些注释是必需的，以便通过 Mypy ` `--strict`` 类型检查。

   此外，上面说明的  :attr:`_orm.declared_attr.directive`  细分成员也是可选的，而且只对  :pep:` 484`  类型工具具有重要意义，因为它调整了在创建方法以覆盖 Declarative 指令时所预期的返回类型。

   .. versionadded:: 2.0 作为 SQLAlchemy ORM 的  :pep:`484`  类型支持的一部分，添加了  :attr:` _orm.declared_attr.directive` ，以区分   :class:`_orm.Mapped`  属性和 Declarative 配置属性。

混合使用和基础类
~~~~~~~~~~~~~~~~~~~

除了使用纯混用外，在这个部分中，这个大多数的技术也可以应用于基类本身的构建上，这些基类的模式应该应用于从特定基础类派生的所有类。下面的例子用了之前一节的 ``Base`` 类来演示一些例子：

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import declared_attr
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        """定义一系列通用元素，这些元素可以应用于映射类
        使用此类作为基类。

        """

        @declared_attr.directive
        def __tablename__(cls) -> str:
            return cls.__name__.lower()

        __table_args__ = {"mysql_engine": "InnoDB"}
        __mapper_args__ = {"eager_defaults": True}

        id: Mapped[int] = mapped_column(primary_key=True)


    class HasLogRecord:
        """标记类，该类与 LogRecord 类之间有一对多关系。"""

        log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

        @declared_attr
        def log_record(self) -> Mapped["LogRecord"]:
            return relationship("LogRecord")


    class LogRecord(Base):
        log_info: Mapped[str]


    class MyModel(HasLogRecord, Base):
        name: Mapped[str]

在上面的示例中，每个 ``MyModel`` 和 ``LogRecord`` 的基类将从它自己中继承所有的 Declarative 指令和列定义。下面是多继承和混用的各个情况无固定的规则。普通的 Python 方法解析规则适用，同时对于以上示例，以下用法同样适用：

    class MyModel(Base, HasLogRecord, CommonMixin):
        name: Mapped[str] = mapped_column()

这对于 Python 方法解析规则来说同样适用，以上面的示例为例，也可以按照下面的方式命名：

..tip::尽管上面的示例使用了基于   :class:`_orm.Mapped`  注释类的   :ref:` Annotated Declarative Table<orm_declarative_mapped_column>`  表单，但 mixin 类也完全可以与非注释和遗留 Declarative 表单混合使用，例如，当直接使用   :class:`_schema.Column`  时，而不是使用   :func:` _orm.mapped_column` 。

.. versionchanged:: 2.0 对于 1.4 系列的 SQLAlchemy 用户，他们可能一直在使用   :ref:`mypy plugin <mypy_toplevel>` ，现在无需使用   :func:` _orm.declarative_mixin`  类装饰器来标记声明性 mixin，只需要不使用 mypy 插件即可。

混入列
~~~~~~~~~~~~~~~~~

在混用情况下，可以在使用   :ref:`declarative table <orm_declarative_table>`  风格的配置时指示列，以便从混用复制到 Declarative 过程生成的   :class:` _schema.Table`  中。   :func:`_orm.mapped_column` 、  :class:` _orm.Mapped`  和   :class:`_schema.Column`  构造中的所有三种形式都可以在声明式 mixin 中进行声明：

    class TimestampMixin:
        created_at: Mapped[datetime] = mapped_column(default=func.now())
        updated_at: Mapped[datetime]


    class MyModel(TimestampMixin, Base):
        __tablename__ = "test"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str]

在上面的示例中，所有包含 ``TimestampMixin`` 的声明式类都将自动包含一个“created_at”列，该列将在所有行插入时应用时间戳，以及一个“updated_at”列，该列不包含默认值，为了示例目的，如果有默认值(有默认值，我们将使用  :paramref:`_schema.Column.onupdate`  参数，该参数被   :func:` _orm.mapped_column`  接受)。 这些列构造在始发 mixin 或基类中始终**从始发 mixin 或基类复制**，以便同一 mixin/base 类可应用于任意数量的目标类，每个目标类都会有自己的列构造。

除了   :class:`_orm.relationship`  之外，Declarative Mixin 的所有形式的 Declarative 表单都受支持，例如，以下 Declarative 列形式都是可以接受的：

* **带注释的属性**——带或不带   :func:`_orm.mapped_column` ::

    class TimestampMixin:
        created_at: Mapped[datetime] = mapped_column(default=func.now())
        updated_at: Mapped[datetime]

* **mapped_column**——带或不带   :class:`_orm.Mapped` ::

    class TimestampMixin:
        created_at = mapped_column(default=func.now())
        updated_at: Mapped[datetime] = mapped_column()

* **Column**——遗留的 Declarative 形式::

    class TimestampMixin:
        created_at = Column(DateTime, default=func.now())
        updated_at = Column(DateTime)

在上述所有形式中，Declarative 处理 mixin 类上的基于列的属性，通过创建**构造的复制品**，该构造的复制品然后应用于目标类。

.. versionchanged:: 2.0 表示配置API现在可以容纳  :class:`_schema.Column`  构造，并且无需使用   :func:` _orm.declared_attr`  即可使用 mixins 使用 ForeignKey 元素的列。以前的限制阻止具有   :class:`_schema.ForeignKey`  元素的列直接在 mixins 中使用，现在已经删除了。

.. _orm_declarative_mixins_relationships:

混合使用关系
~~~~~~~~~~~~~~~~~~~~~~~

由   :func:`~sqlalchemy.orm.relationship`  创建的关系是通过声明性 mixin 类来提供的，它使用了   :class:` _orm.declared_attr`  方法，从而消除了复制关系及其可能绑定列内容所造成的任何歧义。下面是一个示例，演示了如何将一个外键列和一个关系组合在一起，以便两个类，``Foo`` 和 ``Bar``，都可以通过多对一引用公共目标类：

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import declared_attr
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class RefTargetMixin:
        target_id: Mapped[int] = mapped_column(ForeignKey("target.id"))

        @declared_attr
        def target(cls) -> Mapped["Target"]:
            return relationship("Target")


    class Foo(RefTargetMixin, Base):
        __tablename__ = "foo"
        id: Mapped[int] = mapped_column(primary_key=True)


    class Bar(RefTargetMixin, Base):
        __tablename__ = "bar"
        id: Mapped[int] = mapped_column(primary_key=True)


    class Target(Base):
        __tablename__ = "target"
        id: Mapped[int] = mapped_column(primary_key=True)

有了上述的映射，每个 ``Foo`` 和 ``Bar`` 都含有通过 ``.target`` 访问的映射到 ``Target`` 的关系：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> print(select(Foo).join(Foo.target))
    {printsql}SELECT foo.id, foo.target_id
    FROM foo JOIN target ON target.id = foo.target_id{stop}
    >>> print(select(Bar).join(Bar.target))
    {printsql}SELECT bar.id, bar.target_id
    FROM bar JOIN target ON target.id = bar.target_id{stop}

特殊参数，例如  :paramref:`_orm.relationship.primaryjoin` ，也可以在混用类方法中使用，这些类方法通常需要引用正在被映射的类。对于本地映射列需要引用的方案，在普通情况下，这些列的属性是作为   :func:` _orm.declared_attr`  装饰符的属性在声明式类上展示的，所以它们可以用于创建新属性，如下面的例子，将两个列相加：

    from sqlalchemy.orm import column_property
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import declared_attr
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class SomethingMixin:
        x: Mapped[int]
        y: Mapped[int]

        @declared_attr
        def x_plus_y(cls) -> Mapped[int]:
            return column_property(cls.x + cls.y)


    class Something(SomethingMixin, Base):
        __tablename__ = "something"

        id: Mapped[int] = mapped_column(primary_key=True)

在上述示例中，我们可以在使用 ``Something.x_plus_y`` 时产生全表达式：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> print(select(Something.x_plus_y))
    {printsql}SELECT something.x + something.y AS anon_1
    FROM something

.. tip::    :class:`_orm.declared_attr`  装饰符使其被装饰的可调用对象的行为完全像 classmethod 一样。但是，诸如 Pylance_ 这样的类型工具可能无法识别这一点，这有时会导致它在函数体内访问变量 ` `cls`` 时发出警告。要在发现这种情况时解决此问题，可以将 ``@classmethod`` 装饰符与   :class:`_orm.declared_attr`  直接结合使用，如下所示：

      class SomethingMixin:
          x: Mapped[int]
          y: Mapped[int]

          @declared_attr
          @classmethod
          def x_plus_y(cls) -> Mapped[int]:
              return column_property(cls.x + cls.y)

   .. versionadded:: 2.0 作为  :pep:`484`  类型支持的一部分，  :class:` _orm.declared_attr`  可以与 ``@classmethod`` 修饰的函数结合使用，以帮助  :pep:`484`  集成，如有必要。


.. _decl_mixin_inheritance:

使用Mixin类和基类与 映射继承模式
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在处理映射继承模式时，可以使用   :class:`_orm.declared_attr`  在 mixin 类上定义函数，也可以对基类进行增强和未映射的超类，实现一些附加的功能。

在 mixins 或基类上定义由   :class:`_orm.declared_attr`  装饰符装饰的函数时，存在以下重要区别：这些函数生成 Declarative 风格的特殊名称，例如 ` `__tablename__``，``__mapper_args__``，与可能展示为映射属性的函数不同，如   :func:`_orm.mapped_column`  和   :func:` _orm.relationship` 。**定义 Declarative 指令** 的函数被**针对继承层次结构中的每个子类**调用，而为类生成**映射属性** 的函数仅在继承层次结构中的**第一个映射的超类**上调用。

在母类中定义 ``__tablename__`` 函数以生成映射的   :class:`.Table`  名称是一个常见的 mixin 方案，在这种情况下，我们需要注意每个类的**列选项具有这个特征**: 如果我们想在一个除第一个外其它的类中添加一个新的列，我们必须手动为这个类添加列定义，否则 Declarative 映射信息将不完整。 这在下一个部分中说明。

使用   :func:`_orm.declared_attr`  生成特定于每个表的子列和属性

在使用   :class:`_orm.declared_attr`  与   :class:` DeclarativeBase`  类一起使用时，有一种技术可以应用于继承重载列方案，使得在超类里面定义的列属性在子类中也能被复用。 为了将 ``id`` 属性向下分配到子类中，我们可以在下面的 mixin 基本类型中添加一个叫做 ``HasIdMixin`` 的基类：

    class HasIdMixin:
        id: Mapped[int] = mapped_column(primary_key=True)


    class Person(HasIdMixin, Base):
        discriminator: Mapped[str]
        __mapper_args__ = {"polymorphic_on": "discriminator"}


    class Engineer(HasIdMixin, Person):
        id: Mapped[int] = mapped_column(ForeignKey("person.id"), primary_key=True)

        primary_language: Mapped[str]

        __mapper_args__ = {"polymorphic_identity": "engineer"}


    class Manager(Person):
        __mapper_args__ = {"polymorphic_identity": "Manager"}

在上面的示例中，只有 ``Person`` 类接收了一个名叫 ``id`` 的列，映射将在 ``Engineer`` 类上失败，因为它没有定义一个主键。
通常，在联结表继承中，我们希望在每个子类中都有明确定义的列名。 但是在这种情况下，我们可能希望每个表上都有一个名为 ``id`` 的列，并让它们通过外键相互引用。我们可以通过使用  :attr:`.declared_attr.cascading`  修饰符，在实践中为 mixin [前]定义的组及其基类的构造函数提供支持。它指示该函数应该**针对继承层次结构中的每个类**调用，几乎与 ` `__tablename__`` 一样（请参阅下面的警告）。

    class HasIdMixin:
        @declared_attr.cascading
        def id(cls) -> Mapped[int]:
            if has_inherited_table(cls):
                return mapped_column(ForeignKey("person.id"), primary_key=True)
            else:
                return mapped_column(Integer, primary_key=True)


    class Person(HasIdMixin, Base):
        discriminator: Mapped[str]
        __mapper_args__ = {"polymorphic_on": "discriminator"}


    class Engineer(Person):
        primary_language: Mapped[str]
        __mapper_args__ = {"polymorphic_identity": "engineer"}


    class Manager(Person):
        __mapper_args__ = {"polymorphic_identity": "Manager"}

.. warning::

     :attr:`.declared_attr.cascading`  功能目前**不允许**一个子类用不同的函数或值覆盖该属性的值。这是` `@declared_attr`` 被解析的内部方法目前的限制条件，如果检测到这种情况，则会发出警告。此限制仅适用于 ORM 映射列、relationship 和其他   :class:`.MapperProperty`  类型的属性。它**不**适用于 Declarative 指令，如 ` `__tablename__``, ``__mapper_args__`` 等，这些指令的解析方式与 ``(MapperProperty)`` 样式的属性不同。从多个 Mixin 中组合表/映射器参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在使用声明性 Mixin 指定 ``__table_args__`` 或 ``__mapper_args__`` 的情况下，您可能希望将一些参数从几个 Mixin 中与您希望在类本身上定义的参数结合起来。在这里可以使用   :class:`_orm.declared_attr`  装饰器创建用户定义的排序例程，以从多个集合中提取参数::

    from sqlalchemy.orm import declarative_mixin, declared_attr


    class MySQLSettings:
        __table_args__ = {"mysql_engine": "InnoDB"}


    class MyOtherMixin:
        __table_args__ = {"info": "foo"}


    class MyModel(MySQLSettings, MyOtherMixin, Base):
        __tablename__ = "my_model"

        @declared_attr
        def __table_args__(cls):
            args = dict()
            args.update(MySQLSettings.__table_args__)
            args.update(MyOtherMixin.__table_args__)
            return args

        id = mapped_column(Integer, primary_key=True)

使用 Mixin 创建索引
~~~~~~~~~~~~~~~~~~~~

要定义应用于从 Mixin 派生的所有表的命名的、可能是多列的   :class:`.Index` ，可使用 "inline" 形式的   :class:` .Index`  并将其作为 ``__table_args__`` 的一部分建立::

    class MyMixin:
        a = mapped_column(Integer)
        b = mapped_column(Integer)

        @declared_attr
        def __table_args__(cls):
            return (Index(f"test_idx_{cls.__tablename__}", "a", "b"),)


    class MyModel(MyMixin, Base):
        __tablename__ = "atable"
        c = mapped_column(Integer, primary_key=True)

.. _Pylance: https://github.com/microsoft/pylance-release