.. highlight :: pycon+sql

运算符参考
===============================

.. 设置代码，不用显示

    >>> from sqlalchemy import column, select
    >>> from sqlalchemy import create_engine
    >>> engine = create_engine（“sqlite+pysqlite：// /：memory ：”，echo=True）
    >>> from sqlalchemy import MetaData，Table, Column，Integer，String，Numeric
    >>> metadata_obj = MetaData（）
    >>> user_table = Table（
    ...     “user_account”，
    ...     metadata_obj，
    ...     Column（“id”，Integer，primary_key = True），
    ...     Column（“name”，String（30）），
    ...     Column（“fullname”，String），
    ... ）
    >>> from sqlalchemy import ForeignKey
    >>> address_table = Table（
    ...     “address”，
    ...     metadata_obj，
    ...     Column（“id”，Integer，primary_key = True），
    ...     Column（“user_id”，无，ForeignKey（“user_account.id”）），
    ...     Column（“email_address”，String，nullable = False），
    ... ）
    >>> metadata_obj.create_all（engine）
    BEGIN（implicit）
    …
    >>> from sqlalchemy.orm import declarative_base
    >>> Base = declarative_base（）
    >>> from sqlalchemy.orm import relationship
    >>> class User（Base）：
    ...     __tablename__ = “user_account”
    ...
    ...     id = Column（Integer，primary_key = True）
    ...     name = Column（String（30））
    ...     fullname = Column（String）
    ...
    ...     addresses = relationship（“Address”，back_populates = “user”）
    ...
    ...     def __repr__（self）：
    ...         return f“User（id = {self.id！r}，name = {self.name！r}，fullname = {self.fullname！r}）”

    >>> class Address（Base）：
    ...     __tablename__ = “address”
    ...
    ...     id = Column（Integer，primary_key = True）
    ...     email_address = Column（String，nullable = False）
    ...     user_id = Column（Integer，ForeignKey（“user_account.id”））
    ...
    ...     user = relationship（“User”，back_populates = “addresses”）
    ...
    ...     def __repr__（self）：
    ...         return f“Address（id = {self.id！r}，email_address = {self.email_address！r}）”

    >>> conn = engine.connect（）
    >>> from sqlalchemy.orm import Session
    >>> session = Session（conn）
    >>> session.add_all（
    ...     [
    ...         User（
    ...             name = “spongebob ”，
    ...             fullname = “Spongebob Squarepants ”，
    ...             addresses = [Address（email_address = “spongebob @ sqlalchemy.org”）]，
    ...         ），
    ...         User（
    ...             name = “sandy ”，
    ...             fullname = “Sandy Cheeks ”，
    ...             addresses =[
    ...                 Address(email_address="sandy@sqlalchemy.org"),
    ...                 Address(email_address="squirrel@squirrelpower.org"),
    ...             ],
    ...         ),
    ...         User(
    ...             name="patrick",
    ...             fullname="Patrick Star",
    ...             addresses=[Address(email_address="pat999@aol.com")],
    ...         ),
    ...         User(
    ...             name="squidward",
    ...             fullname="Squidward Tentacles",
    ...             addresses=[Address(email_address="stentcl@sqlalchemy.org")],
    ...         ),
    ...         User(name="ehkrabs", fullname="Eugene H. Krabs"),
    ...     ]
    ... ）
    >>> session.commit()
    BEGIN...
    >>> conn.begin()


本节详细介绍了可用于构造SQL表达式的运算符。

这些方法是基于:class：`_sql.Operators`和:class：`_sql.ColumnOperators`基类呈现的。然后可以在子类上使用这些类，包括：

*：class：`_schema.Column`对象

*更一般地来说，是所有Core SQL Expression语言列级别表达式的根：class：`_sql.ColumnElement`对象

*：class：`_orm.InstrumentedAttribute`对象，这是ORM级别映射的属性。

首先，在教程部分中介绍了运算符，包括：

*：doc：`/ tutorial / index`-统一的：术语：`2.0样式`教程

*：doc：`/ orm / tutorial`-ORM：术语：`1.x样式`教程

*：doc：`/ core / tutorial`-Core教程中：术语：`1.x样式`

比较运算符
^^^^^^^^^^^^^^^^^^^^

适用于许多数据类型，包括数字，字符串，日期和许多其他类型的基本比较：

*：meth：`_sql.ColumnOperators.__eq__`（Python“` ==`”操作员）：

    >>> print（column（“x”）==5）
    {printsql}x =：x_1

  ..

*：meth：`_sql.ColumnOperators.__ne__`（Python“`！= `”操作员）：

    >>> print（column（“x”）！=5）
    {printsql}x！=：x_1

  ..

*：meth：`_sql.ColumnOperators.__gt__`（Python“`>`”运算符）：

    >>> print（column（“x”）>5）
    {printsql}x>：x_1

  ..

*：meth：`_sql.ColumnOperators.__lt__`（Python“`<`”操作员）：

    >>> print（column（“x”）<5）
    {printsql}x<：x_1

  ..

*：meth：`_sql.ColumnOperators.__ge__`（Python“`> =`”运算符）：

    >>> print（column（“x”）>=5）
    {printsql}x> =：x_1

  ..

*：meth：`_sql.ColumnOperators.__le__`（Python“`<=`”操作员）：

    >>> print（column（“x”）<=5）
    {printsql}x <=：x_1

  ..

*：meth：`_sql.ColumnOperators.between`：

    >>> print(column("x").between(5, 10))
    {printsql} x BETWEEN :x_1 AND :x_2

  ..

IN比较
^^^^^^^^^^^^^^
SQL IN运算符是SQLAlchemy中的一个主题。由于IN运算符通常针对一组固定值使用，因此SQLAlchemy的绑定参数强制使用两个步骤形成的中间SQL编译字符串，以将其形成为绑定参数的最终列表。换句话说，“它只是工作”。

针对一组值的IN最常见的方法是将其作为列表传递给：meth：`_sql.ColumnOperators.in_`方法::

    >>> print(column("x").in_([1, 2, 3]))
    {printsql}x IN（__[POSTCOMPILE_x_1]__）

特殊绑定形式“__[POSTCOMPILE __”在执行时转换为单个参数，如下所示：

    >>> stmt = select(User.id).where(User.id.in_([1, 2, 3]))
    >>> result = conn.execute(stmt)
    {execsql}SELECT user_account.id
    FROM user_account
    WHERE user_account.id IN（？，？，？）
    [...]（1,2,3）{stop}

空IN表达式
~~~~~~~~~~~~~~~~~~~~

对于空IN表达式，SQLAlchemy通过呈现后端特定的子查询来产生数学上有效的结果，该子查询不返回任何行。换句话说，“它只是工作”::

    >>> stmt = select(User.id).where(User.id.in_([]))
    >>> result = conn.execute(stmt)
    {execsql}SELECT user_account.id
    FROM user_account
    WHERE user_account.id IN（SELECT 1 FROM（SELECT 1）WHERE 1！= 1）
    [...]（）

上述“空集”子查询是正确的并且也以IN运算符的形式呈现，该运算符保持不变。


没有
 

“NOT IN”可通过：meth：`_sql.ColumnOperators.not_in`运算符获得::

    >>> print(column("x").not_in([1, 2, 3]))
    {printsql}（x NOT IN（__[POSTCOMPILE_x_1]__））

这通常更易于将操作员取反与“~”操作员一起使用::

    >>> print（〜column（“x”）in_（【1,2,3】））
    {printsql}（x NOT IN（__[POSTCOMPILE_x_1]__））

元组IN表达式
~~~~~~~~~~~~~~~~~~~~

将元组与元组进行比较是常见的IN用例，因为在将行与一组潜在复合主键值匹配时特别适用。：func：`_sql.tuple_`构造提供了元组比较的基本构建块。然后，：meth：`_sql.Tuple.in_`运算符接收元组列表::

    >>> from sqlalchemy import tuple_
    >>> tup = tuple_(column("x", Integer), column("y", Integer))
    >>> expr = tup.in_([(1, 2), (3, 4)])
    >>> print(expr)
    {printsql}（x，y）IN（__[POSTCOMPILE_param_1]__）

为了说明呈现的参数：

    >>> tup = tuple_(User.id, Address.id)
    >>> stmt = select(User.name).join(Address).where(tup.in_([(1, 1), (2, 2)]))
    >>> conn.execute(stmt).all()
    {execsql}SELECT user_account.name
    FROM user_account JOIN address ON user_account.id = address.user_id
    WHERE（user_account.id，address.id）IN（VALUES（？，？），（？，？））
    [...]（1,1,2,2）{stop}
    [( 'spongebob' ，)（'sandy'，）]

子查询中的IN
~~~~~~~~~~~

最后，：meth：`_sql.ColumnOperators.in_`和：meth：`_sql.ColumnOperators.not_in`运算符可以使用子查询。该形式提供了一个：class：`_sql.Select`构造，直接传递，无需显式转换为命名子查询：

    >>> print(column("x").in_(select(user_table.c.id)))
    {printsql}x IN（SELECT user_account.id
    FROM user_account）

元组按预期工作：

    >>> print(
    ...     tuple_(column("x"), column("y")).in_(
    ...         select(user_table.c.id, address_table.c.id).join(address_table)
    ...     )
    ... )
    {printsql}（x，y）IN（SELECT user_account.id，address.id
    FROM user_account JOIN address ON user_account.id = address.user_id）

恒等比较
^^^^^^^^^^^^^^^^^^^^

这些运算符涉及测试特殊SQL值，例如“`NULL`”，某些数据库支持的布尔常量，例如“true”或“false”：

*：meth：`_sql.ColumnOperators.is_`：

  此运算符将为“x IS y”提供完全的SQL，通常表示为“<expr> IS NULL”。 可以使用常规Python“None”轻松获取“NULL”常量：

    >>> print(column("x").is_(None))
    {printsql}x IS NULL

  SQL NULL也可以明确地使用：func：`_sql.null`构造（如果需要）获得::

    >>> from sqlalchemy import null
    >>> print(column("x").is_(null()))
    {printsql}x IS NULL

  如果在动态值使用，通常不需要显式使用：meth：`_sql.ColumnOperators.is_`，特别是在使用值的情况下，将自动调用它：meth：`_sql.ColumnOperators.__eq__`重载运算符，即« ==`和“None”或：func：`_sql.null`值。 因此，在大多数情况下，不需要使用：meth：`_sql.ColumnOperators.is_`显式使用，特别是当使用动态值时，具有以下语法标记：

    >>> a = None
    >>> print(column("x") == a)
    {printsql}x IS NULL

  请注意，Python“is”操作员**未重载**。即使Python提供了挂钩以重载诸如“==”和“！=”之类的运算符，但它没有提供任何重新定义“is”的方法。

*：meth：`_sql.ColumnOperators.is_not`：

  类似于：meth：`_sql.ColumnOperators.is_`，生成“IS NOT”::

    >>> print(column("x").is_not(None))
    {printsql}x IS NOT NULL

  同样相当于“！= None”：

    >>> print(column("x") != None)
    {printsql}x IS NOT NULL

*：meth：`_sql.ColumnOperators.is_distinct_from`：

  产生SQL IS DISTINCT FROM::

    >>> print(column("x").is_distinct_from("some value"))
    {printsql}x IS DISTINCT FROM：x_1

*：meth：`_sql.ColumnOperators.isnot_distinct_from`：

  产生SQL IS NOT DISTINCT FROM::

    >>> print(column("x").isnot_distinct_from("some value"))
    {printsql}x IS NOT DISTINCT FROM：x_1

字符串比较
^^^^^^^^^^^^^^^^^^

*：meth：`_sql.ColumnOperators.like`::

    >>> print(column("x").like("word"))
    {printsql}x LIKE：x_1

  ..

*：meth：`_sql.ColumnOperators.ilike`：

  不区分大小写的LIKE使用通用后端上的SQL“lower（）”函数。 在PostgreSQL后端上，它将使用“ILIKE”::

    >>> print(column("x").ilike("word"))
    {printsql}lower(x) LIKE lower(:x_1)

  ..

*：meth：`_sql.ColumnOperators.notlike`::

    >>> print(column("x").notlike("word"))
    {printsql}x NOT LIKE：x_1

  ..

*：meth：`_sql.ColumnOperators.notilike`::

    >>> print(column("x").notilike("word"))
    {printsql}lower(x) NOT LIKE lower(:x_1)

  ..

字符串包含
^^^^^^^^^^^^^^^^^^^

字符串包含运算符基本上是由一个LIKE和串联运算符组成的组合，该运算符在大多数后端上是“||”或有时是类似于“concat（）”的函数：

*：meth：`_sql.ColumnOperators.startswith`::

    >>> print(column("x").startswith("word"))
    {printsql}x LIKE：x_1 ||'％'

  ..

*：meth：`_sql.ColumnOperators.endswith`::

    >>> print(column("x").endswith("word"))
    {printsql}x LIKE'％'||：x_1

  ..

*：meth：`_sql.ColumnOperators.contains`::

    >>> print(column("x").contains("word"))
    {printsql}x LIKE'％'||：x_1 ||'％'

  ..

字符串匹配
^^^^^^^^^^^^^^^^^^

匹配操作符始终是特定于后端的，可能在不同的数据库上提供不同的行为和结果：

*：meth：`_sql.ColumnOperators.match`：

  这是一个特定于方言的运算符，如果基础数据库可用，它将使用MATCH功能：

    >>> print(column("x").match("word"))
    {printsql}x 匹配：x_1

  ..

*：meth：`_sql.ColumnOperators.regexp_match`：

  此运算符是特定于方言的。我们可以在PostgreSQL方言中的示例中说明它：

    >>> from sqlalchemy.dialects import postgresql
    >>> print(column("x").regexp_match("word").compile(dialect=postgresql.dialect()))
    {printsql}x ~％（x_1）s

  或者MySQL：

    >>> from sqlalchemy.dialects import mysql
    >>> print(column("x").regexp_match("word").compile(dialect=mysql.dialect()))
    {printsql}x REGEXP％s

  ..

*：meth：`_sql.ColumnOperators.regexp_replace`：

  补充:meth：`_sql.ColumnOperators.regexp_match`，这将针对支持它的后端产生相当于REGEXP REPLACE的效果：

    >>> print(column("x").regexp_replace("foo", "bar").compile(dialect=postgresql.dialect()))
    {printsql}REGEXP_REPLACE(x, %(x_1)s, %(x_2)s)

  ..

*：meth：`_sql.ColumnOperators.collate`：

  提供表示在表达式时间利用特定排序规则的COLLATE SQL运算符：

    >>> print(
    ...     (column("x").collate("latin1_german2_ci") == "Müller").compile(
    ...         dialect=mysql.dialect()
    ...     )
    ... )
    {printsql}(x COLLATE latin1_german2_ci) = %s


  要对文字值使用COLLATE，请使用：func：`_sql.literal`构造：

    >>> from sqlalchemy import literal
    >>> print(
    ...     (literal("Müller").collate("latin1_german2_ci") == column("x")).compile(
    ...         dialect=mysql.dialect()
    ...     )
    ... )
    {printsql}(%s COLLATE latin1_german2_ci) = x

  ..

算术运算符
^^^^^^^^^^^^^^^^^^^^

*：meth：`_sql.ColumnOperators.__add__`，：meth：`_sql.ColumnOperators.__radd__`（Python“` +`”操作员）：

    >>> print(column("x") + 5)
    {printsql}x + :x_1{stop}

    >>> print(5 + column("x"))
    {printsql}:x_1 + x{stop}

  （。）

 请注意，当表达式的数据类型为：class：`_types.String`或类似类型时，：meth：`_sql.ColumnOperators.__add__`运算符将生成：ref：`字符串`concatenation <queryguide_operators_concat_op>`。


*：meth：`_sql.ColumnOperators.__sub__`，：meth：`_sql.ColumnOperators.__rsub__`（Python“`-`”操作员）：

    >>> print(column("x") - 5)
    {printsql}x - ：x_1{stop}

    >>> print(5 - column("x"))
    {printsql}:x_1 - x{stop}

  ..

*：meth：`_sql.ColumnOperators.__mul__`，：meth：`_sql.ColumnOperators.__rmul__`（Python“` *`”操作员）：

    >>> print(column("x") * 5)
    {printsql}x * ：x_1{stop}

    >>> print(5 * column("x"))
    {printsql}:x_1 * x{stop}

  ..

*：meth：`_sql.ColumnOperators.__truediv__`，：meth：`_sql.ColumnOperators.__rtruediv__`（Python“`/`”运算符）.
  这是Python“truediv”运算符，将确保整数真除法发生：

    >>> print(column("x") / 5)
    {printsql}x / CAST（：x_1 AS NUMERIC）{stop}
    >>> print(5 / column("x"))
    {printsql}：x_1 / CAST（x AS NUMERIC）{stop}

  .. versionchanged :: 2.0 Python“/”运算符现在确保执行整数真除法

  ..

*：meth：`_sql.ColumnOperators.__floordiv__`，：meth：`_sql.ColumnOperators.__rfloordiv__`（Python“`//`”操作员）。
  这是Python“floordiv”运算符，将确保进行地板除法。
  对于默认后端以及后端例如PostgreSQL，默认情况下SQL“/”运算符通常以这种方式与整数值交互：

    >>> print(column("x") // 5)
    {printsql}x / ：x_1{stop}
    >>> print(5 // column("x", Integer))
    {printsql}：x_1 / x{stop}

 对于不使用地板除法的后端或与数字值一起使用时，使用FLOOR（）函数以确保进行地板除法：

    >>> print(column("x") // 5.5)
    {printsql}FLOOR（x / ：x_1）{stop}
    >>> print(5 // column("x", Numeric))
    {printsql}FLOOR（：x_1 / x）{stop}

 .. versionadded :: 2.0 支持FLOOR除法

  ..

*：meth：`_sql.ColumnOperators.__mod__`，：meth：`_sql.ColumnOperators.__rmod__`（Python“`％`”操作员）：

    >>> print(column("x")%5)
    {printsql}x％：x_1{stop}
    >>> print(5％column（“x”）)
    {printsql}：x_1％x{stop}

  ..

.. _operators_bitwise：

位运算符
^^^^^^^^^^^^^^^^^

位运算符函数提供了在不同后端上对二进制运算符的统一访问，这些运算符预计在兼容值上运行，例如整数和位字符串（例如，PostgreSQL：class：`_postgresql.BIT`等）。 请注意，这些**不 **是一般布尔运算符。

 .. versionadded :: 2.0.2添加了用于位运算的专用运算符。

*：meth：`_sql.ColumnOperators.bitwise_not`，：func：`_sql.bitwise_not`。 可用作列级方法，产生反向位运算符来针对父对象：

    >>> print(column("x").bitwise_not())
    ~ x

  这个运算符也可用作列表达方式级别的方法，将位运算符应用于单个列表达式：

    >>> from sqlalchemy import bitwise_not
    >>> print(bitwise_not(column("x")))
    ~ x

  ..

*：meth：`_sql.ColumnOperators.bitwise_and`产生按位与运算符::

    >>> print(column("x").bitwise_and(5))
    x&：x_1

  ..

*：meth：`_sql.ColumnOperators.bitwise_or`产生按位或运算符::

    >>> print(column("x").bitwise_or(5))
    x | ：x_1

  ..

*：meth：`_sql.ColumnOperators.bitwise_xor`产生按位异或运算符::

    >>> print(column("x").bitwise_xor(5))
    x ^ ：x_1

  对于PostgreSQL方言，＃用于表示按位异或; 使用这些后端之一时会自动发出：

    >>> from sqlalchemy.dialects import postgresql
    >>> print(column("x").bitwise_xor(5).compile(dialect=postgresql.dialect()))
    x＃％（x_1）s

  ..

*：meth：`_sql.ColumnOperators.bitwise_rshift`，：meth：`_sql.ColumnOperators.bitwise_lshift`
  产生按位移动运算符::

    >>> print(column("x").bitwise_rshift(5))
    x >>：x_1
    >>> print(column("x").bitwise_lshift(5))
    x <<：x_1

  ..


使用连接和否定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最常见的连词“AND”最常见的方法是使用：meth：`_sql.Select.where`方法，以及类似的方法，例如：meth：`_sql.Update.where`和：meth：`_sql.Delete.where`，可以重复使用它。：

    >>> print(select(address_table.c.email_address).where(User.name == "squidward").where(address_table.c.user_id == user_table.c.id))
    {printsql}SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name =：name_1 AND address.user_id = user_account.id

：meth：`_sql.Select.where`，：func：`_sql.Update.where`和：meth：`_sql.Delete.where`还接受多个表达式具有相同的效果：

  >>> print(
  ...     select(address_table.c.email_address).where(
  ...         user_table.c.name == "squidward",
  ...         address_table.c.user_id == user_table.c.id,
  ...     )
  ... )
  {printsql}SELECT address.email_address
  FROM address, user_account
  WHERE user_account.name =：name_1 AND address.user_id = user_account.id

“AND”连词以及其合作伙伴“OR”都可以直接使用：func：`_sql.and_`和：func：`_sql.or_`函数使用：


    >>> from sqlalchemy import and_, or_
    >>> print(
    ...     select(address_table.c.email_address).where(
    ...         and_(
    ...             or_(user_table.c.name == "squidward", user_table.c.name == "sandy"),
    ...             address_table.c.user_id == user_table.c.id,
    ...         )
    ...     )
    ... )
    {printsql}SELECT address.email_address
    FROM address, user_account
    WHERE (user_account.name =：name_1 OR user_account.name =：name_2）
    并且地址.user_id = user_account.id

取消
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

使用：func：`_sql.not_`函数可以进行否定。 这通常将布尔表达式中的运算符反转：

    >>> from sqlalchemy import not_
    >>> print(not_(column("x") == 5))
    {printsql}x！=：x_1

根据需要还可能应用关键字，例如“NOT”::

    >>> from sqlalchemy import Boolean
    >>> print(not_(column("x", Boolean)))
    {printsql} NOT x


结合运算符
^^^^^^^^^^^^^^^^^

以上连接函数：func：`_sql.and_`，：func：`_sql.or_`，
：func：`_sql.not_`也作为重载Python运算符可用：

.. note :: Python“＆”，“|”和“〜”运算符在语言中具有高优先级；因此，对于自身包含表达式的操作数，通常必须应用圆括号，如下面的示例中所示。

*：meth：`_sql.Operators.__and__`（Python“`＆`”操作员）：

  Python二进制“＆”运算符被重载为与：func：`_sql.and_`相同的行为（请注意两个操作数周围的括号）::

     >>> print((column("x") == 5) & (column("y") == 10))
     {printsql} x =：x_1和y =：y_1

  ..

*：meth：`_sql.Operators.__or__`（Python“`|`”运算符）：Python的二元操作符``|``被重载为与 :func:`_sql.or_` 的行为相同（注意两个操作数周围的括号）：

    >>> print((column("x") == 5) | (column("y") == 10))
    {printsql}x = :x_1 OR y = :y_1

* :meth:`_sql.Operators.__invert__` （Python "``~``" 操作符）：

    Python的二元操作符``~``被重载为与 :func:`_sql.not_` 的行为相同，它将翻转现有的操作符，或将``NOT``关键字应用于整个表达式：

    >>> print(~(column("x") == 5))
    {printsql}x != :x_1{stop}

    >>> from sqlalchemy import Boolean
    >>> print(~column("x", Boolean))
    {printsql}NOT x{stop}

.. 安装代码，不进行显示

    >>> conn.close()
    ROLLBACK