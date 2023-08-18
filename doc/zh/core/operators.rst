.. highlight:: pycon+sql

运算符参考
===============================

..  设置代码不会显示

    >>> from sqlalchemy import column, select
    >>> from sqlalchemy import create_engine
    >>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
    >>> from sqlalchemy import MetaData, Table, Column, Integer, String, Numeric
    >>> metadata_obj = MetaData()
    >>> user_table = Table(
    ...     "user_account",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("name", String(30)),
    ...     Column("fullname", String),
    ... )
    >>> from sqlalchemy import ForeignKey
    >>> address_table = Table(
    ...     "address",
    ...     metadata_obj,
    ...     Column("id", Integer, primary_key=True),
    ...     Column("user_id", None, ForeignKey("user_account.id")),
    ...     Column("email_address", String, nullable=False),
    ... )
    >>> metadata_obj.create_all(engine)
    BEGIN (implicit)
    ...
    >>> from sqlalchemy.orm import declarative_base
    >>> Base = declarative_base()
    >>> from sqlalchemy.orm import relationship
    >>> class User(Base):
    ...     __tablename__ = "user_account"
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     name = Column(String(30))
    ...     fullname = Column(String)
    ...
    ...     addresses = relationship("Address", back_populates="user")
    ...
    ...     def __repr__(self):
    ...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"

    >>> class Address(Base):
    ...     __tablename__ = "address"
    ...
    ...     id = Column(Integer, primary_key=True)
    ...     email_address = Column(String, nullable=False)
    ...     user_id = Column(Integer, ForeignKey("user_account.id"))
    ...
    ...     user = relationship("User", back_populates="addresses")
    ...
    ...     def __repr__(self):
    ...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
    >>> conn = engine.connect()
    >>> from sqlalchemy.orm import Session
    >>> session = Session(conn)
    >>> session.add_all(
    ...     [
    ...         User(
    ...             name="spongebob",
    ...             fullname="Spongebob Squarepants",
    ...             addresses=[Address(email_address="spongebob@sqlalchemy.org")],
    ...         ),
    ...         User(
    ...             name="sandy",
    ...             fullname="Sandy Cheeks",
    ...             addresses=[
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
    ... )
    >>> session.commit()
    BEGIN ...
    >>> conn.begin()
    BEGIN ...


本节详细介绍了用于构建 SQL 表达式的运算符。

这些方法按照   :class:`_sql.Operators`  和   :class:` _sql.ColumnOperators`  基类展示。然后，这些方法可以在这些类的子类中使用，包括：

*   :class:`_schema.Column`  对象

* 更一般的   :class:`_sql.ColumnElement`  对象，它们是所有基于 Core SQL 表达式语言的列级表达式的根。

*   :class:`_orm.InstrumentedAttribute`  对象，这是 ORM 级别的映射属性。

这些运算符首先在教程性部分中介绍，包括：

*  :doc:`/tutorial/index`  - 统一教程，采用  :term:` 2.0 风格` 

*  :doc:`/orm/tutorial`  - ORM 教程，采用  :term:` 1.x 风格` 

*  :doc:`/core/tutorial`  - Core 教程，采用  :term:` 1.x 风格` 

比较运算符
^^^^^^^^^^^^^^^^^^^^

适用于许多数据类型（包括数值、字符串、日期和许多其他类型）的基本比较运算符：

*  :meth:`_sql.ColumnOperators.__eq__` （Python "` `==``" 运算符）

  ::

      >>> print(column("x") == 5)
      {printsql}x = :x_1

  ..

*  :meth:`_sql.ColumnOperators.__ne__` （Python "` `!=``" 运算符）

  ::

      >>> print(column("x") != 5)
      {printsql}x != :x_1

  ..

*  :meth:`_sql.ColumnOperators.__gt__` （Python "` `>``" 运算符）

  ::

      >>> print(column("x") > 5)
      {printsql}x > :x_1

  ..

*  :meth:`_sql.ColumnOperators.__lt__` （Python "` `<``" 运算符）

  ::

      >>> print(column("x") < 5)
      {printsql}x < :x_1

  ..

*  :meth:`_sql.ColumnOperators.__ge__` （Python "` `>=``" 运算符）

  ::

      >>> print(column("x") >= 5)
      {printsql}x >= :x_1

  ..

*  :meth:`_sql.ColumnOperators.__le__` （Python "` `<=``" 运算符）

  ::

      >>> print(column("x") <= 5)
      {printsql}x <= :x_1

  ..

*  :meth:`_sql.ColumnOperators.between` 

  ::

      >>> print(column("x").between(5, 10))
      {printsql}x BETWEEN :x_1 AND :x_2

  ..

IN 比较
^^^^^^^^^^^^^^^^^

SQL 的 IN 运算符在 SQLAlchemy 中是一个特殊的话题。由于 IN 运算符通常用于与一组固定值的匹配，因此 SQLAlchemy 的参数绑定强制特性使用一种特殊的 SQL 编译形式，该形式在编译时被渲染为中间形式 SQL 字符串，并以第二步形成最终的绑定的参数列表。换句话说，"它只是工作"。

针对值列表的 IN
~~~~~~~~~~~~~~~~~~~~~~~~~~~

IN 最常见，通常是通过将列表传递给  :meth:`_sql.ColumnOperators.in_`  方法来使用的：

    >>> print(column("x").in_([1, 2, 3]))
    {printsql}x IN (__[POSTCOMPILE_x_1])

在执行时，这里的特殊绑定形式"__[POSTCOMPILE"将呈现为单个参数::

    >>> stmt = select(User.id).where(User.id.in_([1, 2, 3]))
    >>> result = conn.execute(stmt)
    {execsql}SELECT user_account.id
    FROM user_account
    WHERE user_account.id IN (?, ?, ?)
    [...] (1, 2, 3){stop}

空的 IN 表达式
~~~~~~~~~~~~~~~~~~~~

对于空的 IN 表达式，SQLAlchemy 生成一个数学上有效的结果，通过渲染出一种针对特定后端的子查询来返回 0 行。换句话说，"它只是工作"::

    >>> stmt = select(User.id).where(User.id.in_([]))
    >>> result = conn.execute(stmt)
    {execsql}SELECT user_account.id
    FROM user_account
    WHERE user_account.id IN (SELECT 1 FROM (SELECT 1) WHERE 1!=1)
    [...] ()

上面的 "空集" 子查询是正确的，并且是基于保持 IN 运算符不变形渲染出的。

NOT IN
~~~~~~~

"NOT IN" 可以通过  :meth:`_sql.ColumnOperators.not_in`  运算符使用：

    >>> print(column("x").not_in([1, 2, 3]))
    {printsql}(x NOT IN (__[POSTCOMPILE_x_1]))

通常，可以使用 "~" 运算符来否定它：

    >>> print(~column("x").in_([1, 2, 3]))
    {printsql}(x NOT IN (__[POSTCOMPILE_x_1]))

元组 IN 表达式
~~~~~~~~~~~~~~~~~~~~

将元组与元组进行比较是常见的 IN 使用情况，其中包括将行与一组潜在的复合主键值进行匹配的情况。  :func:`_sql.tuple_`  构造为元组比较提供了基本的构建块。然后，  :meth:` _sql.Tuple.in_`   运算符接收元组列表：

    >>> from sqlalchemy import tuple_
    >>> tup = tuple_(column("x", Integer), column("y", Integer))
    >>> expr = tup.in_([(1, 2), (3, 4)])
    >>> print(expr)
    {printsql}(x, y) IN (__[POSTCOMPILE_param_1])

为了说明渲染的参数：

    >>> tup = tuple_(User.id, Address.id)
    >>> stmt = select(User.name).join(Address).where(tup.in_([(1, 1), (2, 2)]))
    >>> conn.execute(stmt).all()
    {execsql}SELECT user_account.name
    FROM user_account JOIN address ON user_account.id = address.user_id
    WHERE (user_account.id, address.id) IN (VALUES (?, ?), (?, ?))
    [...] (1, 1, 2, 2){stop}
    [('spongebob',), ('sandy',)]

子查询 IN
~~~~~~~~~~~

最后，  :meth:`_sql.ColumnOperators.in_`   和  :meth:` _sql.ColumnOperators.not_in`  运算符可以使用子查询。此表单提供直接传递   :class:`_sql.Select`  构造的形式，而不需要显式转换为命名子查询：

    >>> print(column("x").in_(select(user_table.c.id)))
    {printsql}x IN (SELECT user_account.id
    FROM user_account)

元组按预期工作：

    >>> print(
    ...     tuple_(column("x"), column("y")).in_(
    ...         select(user_table.c.id, address_table.c.id).join(address_table)
    ...     )
    ... )
    {printsql}(x, y) IN (SELECT user_account.id, address.id
    FROM user_account JOIN address ON user_account.id = address.user_id)

此运算符提供了一个快捷处理子查询和预设复合运算的功能。


恒等比较
^^^^^^^^^^^^^^^^^^^

这些运算符涉及测试特殊 SQL 值，例如某些数据库支持的 ``NULL``、布尔常量，等等：

*  :meth:`_sql.ColumnOperators.is_` ：

  该运算符将正好提供"``x IS y``"的 SQL 表示形式，通常被视为 "<expr> 是 NULL"。可以使用普通的 Python ``None`` 获得 ``NULL`` 常量：

    >>> print(column("x").is_(None))
    {printsql}x IS NULL

  SQL 中的 NULL 也是显式可用的，如果需要，可以使用   :func:`_sql.null`  构造：

    >>> from sqlalchemy import null
    >>> print(column("x").is_(null()))
    {printsql}x IS NULL

  在与动态值一起使用时，通常不需要显式使用  :meth:`_sql.ColumnOperators.is_` ，特别是在使用  :meth:` _sql.ColumnOperators.__eq__`  连载运算符（例如，"=="）与 ``None`` 或   :func:`_sql.null`  值一起使用时。这样就不需要显式使用  :meth:` _sql.ColumnOperators.is_` ：

    >>> a = None
    >>> print(column("x") == a)
    {printsql}x IS NULL

  请注意，Python ``is`` 运算符**不重载**。即使 Python 提供了钩子以重载运算符例如 ``==`` 和 ``!=``，但它并**没有**提供任何方式来重新定义 ``is``。

*  :meth:`_sql.ColumnOperators.is_not` ：

  类似于  :meth:`_sql.ColumnOperators.is_` ，生成 "IS NOT"：

    >>> print(column("x").is_not(None))
    {printsql}x IS NOT NULL

  等同于 "!= None"：

    >>> print(column("x") != None)
    {printsql}x IS NOT NULL

*  :meth:`_sql.ColumnOperators.is_distinct_from` ：

  生成 SQL IS DISTINCT FROM：

    >>> print(column("x").is_distinct_from("some value"))
    {printsql}x IS DISTINCT FROM :x_1

*  :meth:`_sql.ColumnOperators.isnot_distinct_from` ：

  生成 SQL IS NOT DISTINCT FROM：

    >>> print(column("x").isnot_distinct_from("some value"))
    {printsql}x IS NOT DISTINCT FROM :x_1

字符串比较
^^^^^^^^^^^^^^^^

*  :meth:`_sql.ColumnOperators.like` ：

    >>> print(column("x").like("word"))
    {printsql}x LIKE :x_1

  ..

*  :meth:`_sql.ColumnOperators.ilike` ：

  不区分大小写 LIKE 利用通用后端上的 SQL ``lower()`` 函数。在 PostgreSQL 上，它将使用 ``ILIKE``：

    >>> print(column("x").ilike("word"))
    {printsql}lower(x) LIKE lower(:x_1)

  ..

*  :meth:`_sql.ColumnOperators.notlike` ：

    >>> print(column("x").notlike("word"))
    {printsql}x NOT LIKE :x_1

  ..

*  :meth:`_sql.ColumnOperators.notilike` ：

    >>> print(column("x").notilike("word"))
    {printsql}lower(x) NOT LIKE lower(:x_1)

  ..

字符串包含
^^^^^^^^^^^^^^^^^

许多后端的字符串包含运算符都是建立在 LIKE 和字符串连接运算符的组合上的，后者在大多数后端上是"||"，有时是像 "concat()" 的函数：

*  :meth:`_sql.ColumnOperators.startswith` ：

    >>> print(column("x").startswith("word"))
    {printsql}x LIKE :x_1 || '%'

  ..

*  :meth:`_sql.ColumnOperators.endswith` ：

    >>> print(column("x").endswith("word"))
    {printsql}x LIKE '%' || :x_1

  ..

*  :meth:`_sql.ColumnOperators.contains` ：

    >>> print(column("x").contains("word"))
    {printsql}x LIKE '%' || :x_1 || '%'

  ..

字符串匹配
^^^^^^^^^^^^^^^^^^^

匹配运算符始终是特定于后端的，并且可能在不同的数据库上提供不同的行为和结果：

*  :meth:`_sql.ColumnOperators.match` ：

  该运算符是方言特定的，如果底层数据库支持，则使用底层数据库的 MATCH 功能：

  >>> print(column("x").match("word"))
  {printsql}x MATCH :x_1

*  :meth:`_sql.ColumnOperators.regexp_match` ：

  该操作符是特定于方言的。我们可以用以 PostgreSQL 方言为例：

  >>> from sqlalchemy.dialects import postgresql
  >>> print(column("x").regexp_match("word").compile(dialect=postgresql.dialect()))
  {printsql}x ~ %(x_1)s

  或者是对于 MySQL：

  >>> from sqlalchemy.dialects import mysql
  >>> print(column("x").regexp_match("word").compile(dialect=mysql.dialect()))
  {printsql}x REGEXP %s


  ..

*  :meth:`_sql.ColumnOperators.regexp_replace` ：

  与  :meth:`_sql.ColumnOperators.regexp`  互补，为具有该支持功能的后端生成等价的 REGEX REPLACE：

  >>> print(column("x").regexp_replace("foo", "bar").compile(dialect=postgresql.dialect()))
  {printsql}REGEXP_REPLACE(x, %(x_1)s, %(x_2)s)


  ..

*  :meth:`_sql.ColumnOperators.collate` ：

  生成 COLLATE SQL 运算符，它在表达式时间提供特定的排序方式::

    >>> print(
    ...     (column("x").collate("latin1_german2_ci") == "Müller").compile(
    ...         dialect=mysql.dialect()
    ...     )
    ... )
    {printsql}(x COLLATE latin1_german2_ci) = %s


  要针对文字值使用 COLLATE，请使用   :func:`_sql.literal`  构造::

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

*  :meth:`_sql.ColumnOperators.__add__` ，  :meth:` _sql.ColumnOperators.__radd__`  （Python "``+``" 运算符）：

    >>> print(column("x") + 5)
    {printsql}x + :x_1{stop}

    >>> print(5 + column("x"))
    {printsql}:x_1 + x{stop}

  ..

  注意，当表达式的数据类型为   :class:`_types.String`  或类似的时，：meth:` _sql.ColumnOperators.__add__` 运算符将产生   :ref:`字符串连接 <queryguide_operators_concat_op>` 。

*  :meth:`_sql.ColumnOperators.__sub__` ，  :meth:` _sql.ColumnOperators.__rsub__`  （Python "``-``" 运算符）：

    >>> print(column("x") - 5)
    {printsql}x - :x_1{stop}

    >>> print(5 - column("x"))
    {printsql}:x_1 - x{stop}

  ..

*  :meth:`_sql.ColumnOperators.__mul__` ，  :meth:` _sql.ColumnOperators.__rmul__`  （Python "``*``" 运算符）：

    >>> print(column("x") * 5)
    {printsql}x * :x_1{stop}

    >>> print(5 * column("x"))
    {printsql}:x_1 * x{stop}

  ..

*  :meth:`_sql.ColumnOperators.__truediv__` ，  :meth:` _sql.ColumnOperators.__rtruediv__`  （Python "``/``" 运算符）。
  这是 Python 的 ``truediv`` 运算符，它将确保发生整数真正的分割：

    >>> print(column("x") / 5)
    {printsql}x / CAST(:x_1 AS NUMERIC){stop}
    >>> print(5 / column("x"))
    {printsql}:x_1 / CAST(x AS NUMERIC){stop}

  .. versionchanged:: 2.0  Python 的 ``/`` 运算符现在确保使用整数真正的除法。

  ..

*  :meth:`_sql.ColumnOperators.__floordiv__` ，  :meth:` _sql.ColumnOperators.__rfloordiv__`  （Python "``//``" 运算符）。
  这是 Python 的 ``floordiv`` 运算符，它将确保使用 floor division。对于默认后端以及后端（如 PostgreSQL），SQL ``/`` 运算符通常对整数值进行 floor division。：

    >>> print(column("x") // 5)
    {printsql}x / :x_1{stop}
    >>> print(5 // column("x", Integer))
    {printsql}:x_1 / x{stop}

  对于不使用 floor division 的后端或与数字值一起使用时，将使用 FLOOR() 函数以确保 floor division：

    >>> print(column("x") // 5.5)
    {printsql}FLOOR(x / :x_1){stop}
    >>> print(5 // column("x", Numeric))
    {printsql}FLOOR(:x_1 / x){stop}

  .. versionadded:: 2.0  支持 FLOOR division

  ..

*  :meth:`_sql.ColumnOperators.__mod__` ，  :meth:` _sql.ColumnOperators.__rmod__`  （Python "``%``" 运算符）：

    >>> print(column("x") % 5)
    {printsql}x % :x_1{stop}
    >>> print(5 % column("x"))
    {printsql}:x_1 % x{stop}

  ..

位运算符
^^^^^^^^^^^^^^^^^

按位运算符函数提供了跨不同后端进行位运算符统一访问，这些后端应以兼容的值（例如整数和位字符串（例如 PostgreSQL   :class:`_postgresql.BIT`  等）进行操作。请注意，这些并不是一般的布尔运算符。

.. versionadded:: 2.0.2 为位运算添加了专用运算符。 

*  :meth:`_sql.ColumnOperators.bitwise_not` ，  :func:` _sql.bitwise_not` 。作为一个列级方法，生成对父对象的按位 NOT 语句：

  >>> print(column("x").bitwise_not())
  ~x

  该运算符也可用作列表达式级别的方法，将按位 NOT 应用于单个列表达式：

  >>> from sqlalchemy import bitwise_not
  >>> print(bitwise_not(column("x")))
  ~x

  ..

*  :meth:`_sql.ColumnOperators.bitwise_and`  生成按位 AND：

  >>> print(column("x").bitwise_and(5))
  x & :x_1

  ..

*  :meth:`_sql.ColumnOperators.bitwise_or`  生成 按位 OR：

  >>> print(column("x").bitwise_or(5))
  x | :x_1

  ..

*  :meth:`_sql.ColumnOperators.bitwise_xor`  产生按位异或：

  >>> print(column("x").bitwise_xor(5))
  x ^ :x_1

  对于 PostgreSQL 方言，"#" 用于表示按位 XOR。使用这些后端之一时，会自动出现这样的情况：

  >>> from sqlalchemy.dialects import postgresql
  >>> print(column("x").bitwise_xor(5).compile(dialect=postgresql.dialect()))
  x # %(x_1)s

  ..

*  :meth:`_sql.ColumnOperators.bitwise_rshift` 、  :meth:` _sql.ColumnOperators.bitwise_lshift`  
  产生位移运算符：

  >>> print(column("x").bitwise_rshift(5))
  x >> :x_1
  >>> print(column("x").bitwise_lshift(5))
  x << :x_1

  ..

使用连接和否定
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

最常见的连接符 "AND" 是在我们多次使用  :meth:`_sql.Select.where`  方法时自动应用的，以及类似方法，
如  :meth:`_sql.Update.where`  和  :meth:` _sql.Delete.where` ::

    >>> print(
    ...     select(address_table.c.email_address)
    ...     .where(user_table.c.name == "squidward")
    ...     .where(address_table.c.user_id == user_table.c.id)
    ... )
    {printsql}SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

  :meth:`_sql.Select.where`  ，  :meth:` _sql.Update.where`  和  :meth:`_sql.Delete.where`  还接受多个表达式以达到同样的效果：

    >>> print(
    ...     select(address_table.c.email_address).where(
    ...         user_table.c.name == "squidward",
    ...         address_table.c.user_id == user_table.c.id,
    ...     )
    ... )
    {printsql}SELECT address.email_address
    FROM address, user_account
    WHERE user_account.name = :name_1 AND address.user_id = user_account.id

"And" 连接，以及它的对应的 "OR"，也可使用   :func:`_sql.and_`  和   :func:` _sql.or_`  函数直接使用：

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
    WHERE (user_account.name = :name_1 OR user_account.name = :name_2)
    AND address.user_id = user_account.id

使用   :func:`_sql.not_`  函数可以得到相反的效果。通常会反转布尔表达式中的运算符：

    >>> from sqlalchemy import not_
    >>> print(not_(column("x") == 5))
    {printsql}x != :x_1

当合适时，也可应用关键字，例如 ``NOT``：

    >>> from sqlalchemy import Boolean
    >>> print(not_(column("x", Boolean)))
    {printsql}NOT x


连接操作符
^^^^^^^^^^^^^^^^^^^^^^

上述连接函数   :func:`_sql.and_` ,   :func:` _sql.or_`  和   :func:`_sql.not_`  也可作为重载的 Python 运算符使用：

.. note:: Python 的 "&"、"|" 和 "~" 运算符在语言中具有较高的优先级；因此，必须通常应用用于自身包含表达式的操作数，如下面的示例中所示。

*  :meth:`_sql.Operators.__and__` （Python "` `&``" 运算符）：

  Python 二进制 "&" 运算符被重载为与   :func:`_sql.and_`  表现相同，注意两个操作数周围的圆括号：

     >>> print((column("x") == 5) & (column("y") == 10))
     {printsql}x = :x_1 AND y = :y_1

  ..

*  :meth:`_sql.Operators.__or__` （Python "` `|``" 运算符）：Python二进制“|”运算符被重载以与   :func:`_sql.or_`  相同（请注意两个操作数周围的括号）：

    >>> print((column("x") == 5) | (column("y") == 10))
    {printsql}x = :x_1 OR y = :y_1

*  :meth:`_sql.Operators.__invert__`  (Python "` `~``" operator):

Python二进制“~”运算符被重载以与   :func:`_sql.not_`  相同，有时候用作翻转现有运算符，或者将"NOT"关键字应用于整个表达式中：

    >>> print(~(column("x") == 5))
    {printsql}x != :x_1{stop}

    >>> from sqlalchemy import Boolean
    >>> print(~column("x", Boolean))
    {printsql}NOT x{stop}

.. 设置代码，不显示

    >>> conn.close()
    ROLLBACK