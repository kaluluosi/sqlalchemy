更改属性行为
===========================
本章将讨论用于修改ORM映射属性行为的功能和技术，包括使用：func:`_orm.mapped_column`，func:`_orm.relationship`等映射的属性。

简单验证器
-----------------
向属性添加“验证”程序的快速方法是使用：func:`~sqlalchemy.orm.validates` 装饰器。属性验证器可能抛出异常，停止变异属性的进程，也可能将给定的值更改为不同的值。像所有属性扩展一样，验证器仅由普通用户代码调用；当ORM填充对象时，不会发出它们。

```python
    from sqlalchemy.orm import validates

    class EmailAddress(Base):
        __tablename__ = "address"

        id = mapped_column(Integer, primary_key=True)
        email = mapped_column(String)

        @validates("email")
        def validate_email(self, key, address):
            if "@" not in address:
                raise ValueError("failed simple email validation")
            return address
```

对于添加到集合中的项目，验证器接收集合追加事件。

```python
    from sqlalchemy.orm import validates

    class User(Base):
        # ...

        addresses = relationship("Address")

        @validates("addresses")
        def validate_address(self, key, address):
            if "@" not in address.email:
                raise ValueError("failed simplified email validation")
            return address
```

默认情况下，验证函数不会为集合删除事件发出，因为典型期望是删除值不需要验证。但是，：func:`.validates`支持通过指定''Include_removes=True``到装饰器来接收这些事件。当设置此标志时，验证函数必须接收一个额外的布尔参数，如果为True，则表示该操作是删除操作。

```python
    from sqlalchemy.orm import validates

    class User(Base):
        # ...

        addresses = relationship("Address")

        @validates("addresses", include_removes=True)
        def validate_address(self, key, address, is_remove):
            if is_remove:
                raise ValueError("not allowed to remove items from the collection")
            else:
                if "@" not in address.email:
                    raise ValueError("failed simplified email validation")
                return address
```

当通过backref链接相互关联的验证程序之间存在的情况时，也可以进行定制，使用''Include_backrefs=False''选项，当设置为''False''时，该选项避免验证函数在由backref导致事件发生时显示。

```python
    from sqlalchemy.orm import validates

    class User(Base):
        # ...

        addresses = relationship("Address", backref="user")

        @validates("addresses", include_backrefs=False)
        def validate_address(self, key, address):
            if "@" not in address:
                raise ValueError("failed simplified email validation")
            return address
```

请注意，：func:`~.validates`装饰器是建立在属性事件之上的便利函数。需要在属性更改行为的配置上具有更多控制的应用程序可以使用此系统，查看：class:`~.AttributeEvents`。

使用自定义数据类型在核心级别
-----------------------------------------------------------

用于以适当的方式在Python中表示数据与表示数据库中的数据之间转换数据的非ORM方法可以通过使用应用于映射的:_schema.Table元数据的自定义数据类型来实现。这在编码/解码某些样式的情况下更为常见，这些样式既在数据流向数据库时也在数据返回时发生；请查看Core文档中的：ref:`types_typedecorator`。

使用解析器和混合体
-----------------------------------------------------------

一种更为全面的方法是使用：term:`descriptors`生成属性的修改行为。使用``property()``函数在Python中常规使用描述符的方法。描述符的标准SQLAlchemy技术是创建普通描述符，并从另一个名称处的映射属性进行读写。下面我们使用Python 2.6样式的属性说明了这一点：

```python
    class EmailAddress(Base):
        __tablename__ = "email_address"

        id = mapped_column(Integer, primary_key=True)

        # name the attribute with an underscore,
        # different from the column name
        _email = mapped_column("email", String)

        # then create an ".email" attribute
        # to get/set "._email"
        @property
        def email(self):
            return self._email

        @email.setter
        def email(self, email):
            self._email = email
```

以上方法有效，但我们可以添加更多内容。虽然我们的``EmailAddress``对象将值通过``email``描述符传递到``_email``映射属性中，但该类级别``EmailAddress.email``属性不具有可与``_sql.Select``一起使用的常规表达式语义。为了提供这些语义，我们使用以下对：mod:`~sqlalchemy.ext.hybrid`扩展：

```python
    from sqlalchemy.ext.hybrid import hybrid_property

    class EmailAddress(Base):
        __tablename__ = "email_address"

        id = mapped_column(Integer, primary_key=True)

        _email = mapped_column("email", String)

        @hybrid_property
        def email(self):
            return self._email

        @email.setter
        def email(self, email):
            self._email = email
```

除了在拥有``EmailAddress``实例时提供getter/setter行为之外，``.email``属性在使用类级别时，也提供了SQL表达式，即，在直接从``EmailAddress``类中使用：

```sqlalchemy
    from sqlalchemy.orm import Session
    from sqlalchemy import select

    session = Session()

    address = session.query(EmailAddress).filter(EmailAddress.email == "address@example.com").one()
    address.email = "otheraddress@example.com"
    session.commit()
```

: class:`~.hybrid_property`还允许我们更改属性的行为，包括在实例级别和类/表达式级别访问属性时，使用：meth:`.hybrid_property.expression`修饰符定义不同的行为。例如，如果我们想自动添加主机名，我们可能会定义两组字符串操作逻辑：

```python
    class EmailAddress(Base):
        __tablename__ = "email_address"

        id = mapped_column(Integer, primary_key=True)

        _email = mapped_column("email", String)

        @hybrid_property
        def email(self):
            """Return the value of _email up until the last twelve
            characters."""

            return self._email[:-12]

        @email.setter
        def email(self, email):
            """Set the value of _email, tacking on the twelve character
            value @example.com."""

            self._email = email + "@example.com"

        @email.expression
        def email(cls):
            """Produce a SQL expression that represents the value
            of the _email column, minus the last twelve characters."""

            return func.substr(cls._email, 0, func.length(cls._email) - 12)
```

上面，访问``EmailAddress``实例的``email``属性将返回``_email``属性的值，从该值中删除或添加主机名``@example.com``。当我们针对``email``属性查询时，将渲染生成相同效果的SQL函数：

```sqlalchemy
    address = session.query(EmailAddress).filter(EmailAddress.email == "address").one()
```

请查看：ref:`hybrids_toplevel`的Hybrids。

同义词
----------------------
同义词是映射器级别的构造，允许类上的任何属性“镜像”与映射的其他属性。从最基本的角度来看，同义词是一种通过其他名称提供某个属性的简单方式

```python
    from sqlalchemy.orm import synonym

    class MyClass(Base):
        __tablename__ = "my_table"

        id = mapped_column(Integer, primary_key=True)
        job_status = mapped_column(String(50))

        status = synonym("job_status")
```

上面的 `` MyClass``类有两个属性“job_status”和“status”将作为一个属性进行运行，在表达式级别精确地实现：

```python
    >>> print(MyClass.job_status == "some_status")
    {printsql}my_table.job_status = :job_status_1{stop}

    >>> print(MyClass.status == "some_status")
    {printsql}my_table.job_status = :job_status_1{stop}
```
和在实例级别：

```python
    >>> m1 = MyClass(status="x")
    >>> m1.status, m1.job_status
    ('x', 'x')

    >>> m1.job_status = "y"
    >>> m1.status, m1.job_status
    ('y', 'y')
```
:func:`.synonym`可以用于任何种类的映射属性，该属性是的子类:class:`.MapperProperty`，包括映射的列和关系，以及同义词本身。

:class:`.PropComparator`子类可以作为``comparator_factory``参数传递到: func:`.column_property`，func:`_orm.relationship`和:func:`.composite`等ORM级函数中，从而为属性行定义提供操作符重新定义。在此级别上自定义运算符的用例是罕见的。请参阅：class:`~.PropComparator`的文档进行概述。

自定义比较器
----------------------

由SQLAlchemy ORM和Core表达式语言使用的“运算符”是完全可定制的。例如，比较表达式``User.name =='ed'``使用了构建在Python的操作符``operator.eq``之上，SQLAlchemy实际上将该操作符所关联的实际SQL构造进行了修改。新的操作还可以与列表达式一起关联。对于列表达式所执行的运算符最直接地在类型级别进行重新定义 - 请参见该部分types_operators的说明。

像：func:`.column_property`，：func:`_orm.relationship`，和：func:`.composite` ORM级函数也会在每个函数的``comparator_factory``参数中传递:class:`.PropComparator`子类来重新定义运算符。在这个级别上自定义运算符的使用用例是罕见的。请参阅:class:`.PropComparator`文档进行概述。