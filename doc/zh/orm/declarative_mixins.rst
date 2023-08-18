使用 Mixins 构成映射层次
======================

使用 :ref: `声明式 <orm_declarative_mapping>` 风格来映射类时，有一个常见的需求，就是跨类共享
常见的功能，例如特定的列、表或映射器选项，命名方案或其他映射的属性。当使用声明式映射时，可以通过使用
:meth: `申明属性(dec_属性< 函数 <搜寻结构脚本 >])` 类和通过增加声明基类本身来支持这种语言习惯。

.. tip :: 除了 mixin 类之外，还可以使用 :pep: `593`“已注解”类型共享许多类之间的常见列选项。
  有关这些 SQLAlchemy 2.0 功能的背景，请参见下面的相关章节。
  
例如，下面是一些常见混合语言的例子:: 

   from sqlalchemy import ForeignKey
    from sqlalchemy.orm import declared_attr
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class CommonMixin:
        """将此类用作 mixin 类应用于映射类时，定义了一系列通用元素。"""

        @declared_attr.directive
        def __tablename__(cls) -> str:
            return cls.__name__.lower()

        __table_args__ = {"mysql_engine": "InnoDB"}
        __mapper_args__ = {"eager_defaults": True}

        id: Mapped[int] = mapped_column(primary_key=True)


    class HasLogRecord:
        """标记具有对“LogRecord”类的多对一关系的类。"""

        log_record_id: Mapped[int] = mapped_column(ForeignKey("logrecord.id"))

        @declared_attr
        def log_record(self) -> Mapped["LogRecord"]:
            return relationship("LogRecord")


    class LogRecord(CommonMixin, Base):
        log_info: Mapped[str]


    class MyModel(CommonMixin, HasLogRecord, Base):
        name: Mapped[str]
      
上面的示例说明了一个名为“ MyModel”的类，其中包括两个 mixin “CommonMixin”和“ HasLogRecord”
作为其基础，以及一个辅助类“ LogRecord”，
该辅助类还包括“ CommonMixin”，展示了有许多结构支持了 mixin 和基类，包括：

* 使用 :meth:`_orm.mapped_column`, :类:`~_orm.Mapped`或 :类:`~_schema.Column` 声明的列从mixin或基类复制到将要映射的目标类中
列属性“CommonMixin.id”和“ HasLogRecord.log_record_id”通过上述方法进行了说明。
* 混合或基类本身可以分配声明性指令，例如``__table_args__`` 和 ``__mapper_args__``，其中它们会自动
适用于从混合或基类继承的任何类。 上面的例子使用 ``__table_args__`` 和 ``__mapper_args__`` 属性进行说明。
* 所有 Declarative 指令，包括所有 ``__tablename__``、``__table__``、``__table_args__`` 和 ``__mapper_args__`` 指令，
可以使用用户定义的类方法来实现，这些类方法装饰了 :类:`_orm.declared_attr` 装饰符(特别的是
 :attr: `_orm.declared_attr.directive` 子成员, 稍后将在详细讨论) 。
这在上面的例子中使用一个``def __tablename__(cls)``类方法，该方法动态生成了一个 :类:`.Table`名称。
当应用于" MyModel "类时，表名将被生成为 "mymodel "，而当应用于 "LogRecord "类时，表名将被生成为 "logrecord"。
* 其他 ORM 属性（例如 :功能：`_orm.relationship`）可以使用用户定义的类方法生成到映射的目标类中，
也用由带有 :类:`_orm.declared_attr` 装饰符的用户定义的类方法显示。 上面，这是
通过生成多对一 :func:`_orm.relationship` 到名为“ LogRecord ”的映射对象来说明的。
 
上述功能都可以使用 :func:`_sql.select` 示例演示：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import select
    >>> print(select(MyModel).join(MyModel.log_record))
    {print sql}SELECT mymodel.name, mymodel.id, mymodel.log_record_id
    FROM mymodel JOIN logrecord ON logrecord.id = mymodel.log_record_id

.. tip :: :类:`_orm.declared_attr` 的示例将尝试提供每个方法示例的 :PEP:`484` 注释。
. :类:`_orm.declared_attr` 函数的注释是完全可选的，并且不
为 Declarative 消费; 但是，必须使用这些注释，以便
通过 Mypy ``-- strict`` 类型检查。

  并且 :attr:`_orm.declared_attr.directive` 子成员上面也是可选的，仅适用于 :PEP:`484` 类型工具，
  它在创建方法以覆盖声明式指令时调整预期的返回类型方面。 包括， 示例上面的
  使用“ _orm.declared_attr.directive ”实例说明在 :pep: `484` 中的类型支持时添加了 :版本添加:2.0>`ORM SQLAlchemy`，以区分 :类:` Mapper `属性和声明性配置性属性。
 
在混合和基类中添加映射器属性
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

与许多执行类具有相同功能的手段（例如列、表和映射器选项、命名计划或其他映射属性） 的行为类似的
进化过程，任何映射到“ _orm.declared_attr”中装饰的函数的mixed-in属性都将被设计为
加入到基本类或任何由基本类衍生的其他目标类的元素中，因此任何这种构造都可以应用于一组类。

如下所示，任何使用混合的模式都可以表示为类的本机继承：

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import declared_attr
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column


    class Base(DeclarativeBase):
        pass


    class Mixin:
        column_a: Mapped[str] = mapped_column()
        column_b: Mapped[str] = mapped_column()


    class A(Mixin, Base):
        column_c: Mapped[str] = mapped_column()


由于继承的特点，再次声明“ column_a”，“ column_b”即可在生成的子类上覆盖它们。
此外，对于任何前面的混合体，它们都将将其余部分保留在新目标类中。任何列或其它输出，例如“ column_c”，
都是将基类和任何相关派生类的元素合并在一起。

.. versionchanged :: 1.0.0：这是标准行为，但只在 1.0.0 后才被正式记录。



...从多个Mixin组合Table/Mapper参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在使用声明式mixin指定``__table_args__``或``__mapper_args__``时，您可能希望将一些参数从多个mixin与您希望在类本身上定义的参数组合在一起。这里可以使用
:class:`_orm.declared_attr`装饰器创建用户定义的收集程序，从多个集合中获取值。

.. sourcecode:: python

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

使用Mixin创建索引
~~~~~~~~~~~~~~~~~

为了定义一个命名的，可能是多列的:class:`.Index`，它应用于从Mixin派生的所有表，可以使用``__table_args__``将其作为“inline”形式的:class:`.Index`之一来确定：

.. sourcecode:: python

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