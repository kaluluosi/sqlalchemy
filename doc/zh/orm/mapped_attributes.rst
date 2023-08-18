.. _mapping_attributes_toplevel:

.. currentmodule:: sqlalchemy.orm

更改属性行为
===========================

本节将讨论修改ORM映射属性的特性和技术，包括使用   :func:`_orm.mapped_column` 、  :func:` _orm.relationship`  和其他方法映射的属性。

.. _simple_validators:

简单的验证器
-----------------

将"验证"过程添加到属性的快捷方式是使用   :func:`~sqlalchemy.orm.validates`  装饰器。属性验证器可以引发异常，停止改变属性的值的处理，或者可以将给定值更改为不同的值。像所有属性扩展一样，验证器仅由正常的用户代码调用；在ORM填充对象时不会发出。

::

    from sqlalchemy.orm import validates


    class EmailAddress(Base):
        __tablename__ = "address"

        id = mapped_column(Integer, primary_key=True)
        email = mapped_column(String)

        @validates("email")
        def validate_email(self, key, address):
            if "@" not in address:
                raise ValueError("simplr email validation failed")
            return address

验证器还接收集合附加事件的，当向集合中添加项目时会触发事件::

    from sqlalchemy.orm import validates


    class User(Base):
        # ...

        addresses = relationship("Address")

        @validates("addresses")
        def validate_address(self, key, address):
            if "@" not in address.email:
                raise ValueError("failed simple email validation")
            return address

默认情况下，验证函数不会在集合删除事件中发出，因为通常的期望是被丢弃的值不需要经过验证。 但是，通过将``include_removes=True``指定给装饰器， :func:`.validates` 支持接收这些事件。当设置了此标志时，在删除操作上验证函数必须接收一个附加的布尔参数，如果为True则表示操作是删除操作:

::

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

通过自反引用后另一个验证器相关的情况也可以定制。可以使用``include_backrefs=False``选项来使用此选项，当设置为``False``时，防止在事件作为反向引用的结果出现时发出验证器函数：

::

    from sqlalchemy.orm import validates


    class User(Base):
        # ...

        addresses = relationship("Address", backref="user")

        @validates("addresses", include_backrefs=False)
        def validate_address(self, key, address):
            if "@" not in address:
                raise ValueError("failed simplified email validation")
            return address

上面，如果我们像``some_address.user = some_user``那样给``Address.user``赋值，
将不会产生``validate_address()``函数，即使向``some_user.addresses``添加一个append事件，事件是由反向引用引起的。

请注意：  :func:`~.validates`  装饰器是建立在属性事件基础之上的一种便捷函数。需要更多地控制属性更改行为的应用程序可以利用这个系统在   :class:` ~.AttributeEvents`  中描述。

.. autofunction:: validates

在核心级别使用自定义数据类型
-----------------------------------------

在不使用ORM的情况下，可以通过使用应用到映射   :class:`_schema.Table`  元数据的自定义数据类型以适合转换Python和数据库中表示的数据的方式来影响列的值。这在某种编码/解码样式的情况下更加常见，因为数据在传递到数据库和从数据库返回时都会进行转换。有关更多信息，请参阅Core文档中的   :ref:` types_typedecorator` 。

.. _mapper_hybrids:

使用描述符和混合（Hybrids）
-----------------------------

更全面的一种方法是使用  :term:`descriptors`  来为属性生成修改后的行为。这些通常在Python中使用property()函数。描述符的标准SQLAlchemy技术是创建一个简单的描述符，并通过使用不同的名称从映射的属性读写。以下是使用Python 2.6风格属性的示例::

    class EmailAddress(Base):
        __tablename__ = "email_address"

        id = mapped_column(Integer, primary_key=True)

        # 将属性命名为带有下划线的名称，不同于列名
        _email = mapped_column("email", String)

        # 然后创建一个".email"属性以获取/设置"._email"
        @property
        def email(self):
            return self._email

        @email.setter
        def email(self, email):
            self._email = email

上面的方法是有效的，但我们还可以添加更多内容。虽然我们的``EmailAddress``对象将通过``email``描述符将值传递到``_email``映射的属性中，但类级别的``EmailAddress.email``属性没有使用   :class:`_sql.Select`  可用的常规表达式语义。为了提供这些语义，我们改为使用  :mod:` ~sqlalchemy.ext.hybrid`  扩展，如下所示::

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

除了为我们有一个``EmailAddress``实例提供getter/setter行为外，``.email``属性还在类级别时提供SQL表达式直接使用，也就是从``EmailAddress``类直接使用:

.. sourcecode:: python+sql

    from sqlalchemy.orm import Session
    from sqlalchemy import select

    session = Session()

    address = session.scalars(
        select(EmailAddress).where(EmailAddress.email == "address@example.com")
    ).one()
    {execsql}SELECT address.email AS address_email, address.id AS address_id
    FROM address
    WHERE address.email = ?
    ('address@example.com',)
    {stop}

    address.email = "otheraddress@example.com"
    session.commit()
    {execsql}UPDATE address SET email=? WHERE address.id = ?
    ('otheraddress@example.com', 1)
    COMMIT
    {stop}

: class:`~.hybrid_property` 还允许我们更改属性的行为，包括使用  :meth:`.hybrid_property.expression`  修饰符定义在实例级别和类/表达级别访问属性时的不同行为。例如，如果我们想自动添加主机名，我们可以定义两组字符串操作逻辑:

::

    class EmailAddress(Base):
        __tablename__ = "email_address"

        id = mapped_column(Integer, primary_key=True)

        _email = mapped_column("email", String)

        @hybrid_property
        def email(self):
            """返回邮箱地址值中最后12个字符之前的值"""

            return self._email[:-12]

        @email.setter
        def email(self, email):
            """设置邮件地址值，将12个字符的值@example.com拼接在后面"""

            self._email = email + "@example.com"

        @email.expression
        def email(cls):
            """生成一种SQL表达式，表示_email列的值，减去了最后12个字符的长度"""

            return func.substr(cls._email, 0, func.length(cls._email) - 12)

上述示例中，在访问``EmailAddress``实例的``email``属性时，将返回``_email``属性的值，从中删除或添加邮箱地址。当我们针对``email``属性查询时，将呈现一个SQL函数，该函数产生相同的效果:

.. sourcecode:: python+sql

    address = session.scalars(
        select(EmailAddress).where(EmailAddress.email == "address")
    ).one()
    {execsql}SELECT address.email AS address_email, address.id AS address_id
    FROM address
    WHERE substr(address.email, ?, length(address.email) - ?) = ?
    (0, 12, 'address')
    {stop}

在更复杂的情况下，使用   :ref:`hybrid attribute <mapper_hybrids>`  功能更好，因为它更倾向于Python描述符。严格来说，  :func:` .synonym` .hybrid_property` 所能做到的所有事情，因为它也支持自定义SQL功能的注入，但是混合属性在更复杂的情况下更容易使用。

.. autofunction:: validates

检验（Synonyms）
------------------------

检验是一个映射层面的构造，允许您将类上的任何属性“镜像”到映射的另一个属性上。

在最基本的意义上，同义词是一种简单的方式，通过附加名称来使一个特定的属性可用::

    from sqlalchemy.orm import synonym


    class MyClass(Base):
        __tablename__ = "my_table"

        id = mapped_column(Integer, primary_key=True)
        job_status = mapped_column(String(50))

        status = synonym("job_status")

上述类 ``MyClass`` 有两个属性，``.job_status`` 和 ``.status ``，在表达式级别上表现为一个属性:

.. sourcecode:: pycon+sql

    >>> print(MyClass.job_status == "some_status")
    {printsql}my_table.job_status = :job_status_1{stop}

    >>> print(MyClass.status == "some_status")
    {printsql}my_table.job_status = :job_status_1{stop}

在实例级别和在实例级别表达式中都表现为一个属性::

    >>> m1 = MyClass(status="x")
    >>> m1.status, m1.job_status
    ('x', 'x')

    >>> m1.job_status = "y"
    >>> m1.status, m1.job_status
    ('y', 'y')

对于任何通过   :class:`.MapperProperty`  子类进行了映射的属性，包括映射的列和关系，除了同义词本身， :func:` .synonym`也可以用于任何映射属性。


除了简单反映之外，  :func:`.synonym` 。我们可以为我们的` `status``同义词提供一个``@property``描述符::

    class MyClass(Base):
        __tablename__ = "my_table"

        id = mapped_column(Integer, primary_key=True)
        status = mapped_column(String(50))

        @property
        def job_status(self):
            return "Status: " + self.status

        job_status = synonym("status", descriptor=job_status)

使用Declarative时，可以使用   :func:`.synonym_for`  装饰器更简洁地表示上述模式::

    from sqlalchemy.ext.declarative import synonym_for


    class MyClass(Base):
        __tablename__ = "my_table"

        id = mapped_column(Integer, primary_key=True)
        status = mapped_column(String(50))

        @synonym_for("status")
        @property
        def job_status(self):
            return "Status: " + self.status

虽然   :func:`.synonym`  对于简单的反映是有用的，但是使用混合属性特性更好地处理使用描述符增强属性行为的用例，后者更适用于Python描述符。严格来说，  :func:` .synonym`  可以做到   :class:`.hybrid_property`  所能做到的一切，因为它还支持自定义SQL功能的注入，不过需要在更复杂情况下使用混合。

.. autofunction:: synonym

.. _custom_comparators:

运算符自定义
----------------------

SQLAlchemy ORM和Core表达式语言使用的“运算符”是完全可定制的。例如，比较表达式``User.name == 'ed'`` 使用一个内置的Python运算符：``operator.eq``-实际上SQLAlchemy将这样的运算符关联的SQL结构可以被修改。也可以将新操作与列表达式关联。作用于列表达式的操作最直接地在类型级别重新定义 - 请参阅   :ref:`types_operators`  部分的说明。

由   :func:`.column_property` ,   :func:` _orm.relationship`  和   :func:`.composite`  等ORM级函数也提供操作重定义，在每个函数中将   :class:` .PropComparator`  子类传递给 ``comparator_factory`` 参数。这个级别上的操作符自定义功能使用很少。 在   :class:`.PropComparator`  的文档中概述了操作符在此级别上的自定义。