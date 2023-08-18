SQL表达式
===============

.. contents::
    :local:
    :class: faq
    :backlinks: none

.. _faq_sql_expression_string:

如何将SQL表达式呈现为字符串，可能带有联接的参数？
------------------------------------------------------------------------------------

大多数情况下，将SQLAlchemy Core语句对象或表达式片段以及ORM :class:`_query.Query`对象“字符串化”，可以简单地使用“str（）”内置函数，如下所示：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import table, column, select
    >>> t = table("my_table", column("x"))
    >>> statement = select(t)
    >>> print(str(statement))
    {printsql}SELECT my_table.x
    FROM my_table

可以对ORM :class:`_query.Query` 对象以及任何语句（例如:func:`_expression.select`, :func:`_expression.insert`等）进行调用。常用的表达式片段如下：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import column
    >>> print(column("x") == "some value")
    {printsql}x = :x_1

具有特定数据库字符串格式的功能或一些特定数据库中才能使用的元素时，stringify这些状态或片段变得复杂。这种情况下，我们可能会得到不正确的语法甚至是:class:`.UnsupportedCompilationError`异常。解决方法是使用:meth:`_expression.ClauseElement.compile`方法的方法，同时传递一个代表目标数据库的：class:`_engine.Engine`或:class:`.Dialect`对象。例如下面的例子，如果我们使用MySQL数据库，请将语句字符串化：

    from sqlalchemy import create_engine

    engine = create_engine("mysql+pymysql://scott:tiger@localhost/test")
    print(statement.compile(engine))

更直接的方法是，不用构建: class:`_engine.Engine`对象，直接实例化:class:`.Dialect`对象，例如我们使用PostgreSQL分支：

    from sqlalchemy.dialects import postgresql

    print(statement.compile(dialect=postgresql.dialect()))

请注意，任何dialect都可以使用 :func:`_sa.create_engine`本身来组装，使用虚拟URL，然后访问: attr:`_engine.Engine.dialect`属性，例如，假设我们想要一个psycopg2的dialect对象：

    e = create_engine("postgresql+psycopg2://")
    psycopg2_dialect = e.dialect

对于给定的ORM :class:`~.orm.query.Query` 对象，要获得
:meth:`_expression.ClauseElement.compile`
方法，我们只需要首先访问:attr:`~.orm.query.Query.statement`访问器即可：

    statement = query.statement
    print(statement.compile(someengine))

渲染联接参数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. warning:: **永远不要**使用这些技术与来自于不受信任输入的字符串内容，比如来自于网络表单或其他用户输入的应用程序。在计算非DDL SQL语句时，SQLAlchemy的将Python值转换为直接SQL字符串值的功能是**不安全的，不经过任何类型的数据验证**。 对于针对关系数据库的非DDL SQL语句编程调用时，始终使用绑定参数。

上面的方法将呈现SQL语句作为传递给Python：term：`DBAPI`，这包括联接的参数不会呈现为联接。SQLAlchemy通常不会执行字符串联接参数操作，因为这是通过Python DBAPI适当处理的，更不用说绕过绑定参数可能是现代网络应用程序中最广泛利用的安全漏洞。SQLAlchemy的一部分功能可以在某些情况下执行此类字符串串联操作，例如DDL的发射。为了访问此功能，我们可以使用“literal_binds”标志传递给“compile_kwargs”::

    from sqlalchemy.sql import table, column, select

    t = table("t", column("x"))
    s = select(t).where(t.c.x == 5)

    # **对不受信任的输入请勿使用!!!**
    print(s.compile(compile_kwargs={"literal_binds": True}))

    # 渲染具体分支
    print(s.compile(dialect=dialect, compile_kwargs={"literal_binds": True}))

    # 或者，如果有引擎，请作为第一个参数传递
    print(s.compile(some_engine, compile_kwargs={"literal_binds": True}))

此功能主要用于记录或调试目的，其中查询的原始 SQL 字符串可能被证明是有用的。

以上方法的注意事项是它仅支持基本类型，例如整数和字符串，而且另外，如果直接使用一个无预设值的:func:`.bindparam`，它也无法对这个类型的链接字符串操作。无条件地对所有参数字符串操作的方法在下面给出。

.. 提示::

   SQLAlchemy 不支持所有数据类型的完整字符串化有三个原因:

   1. 当DBAPI正常使用时，这是一个已得到支持的功能。SQLAlchemy项目不能确保为所有后端重复这种功能，以及这种冗余工作通常会导致显著的测试和持续支持开销。

   2. 对于特定的数据库，已经提供了将绑定参数联接为完整字符串的功能，这表明将这些完整字符串传递到数据库执行是不必要和不安全的。 SQLAlchemy不希望以任何方式鼓励这种用途。

   3. 渲染文本值的区域是最可能报告安全问题的区域。 SQLAlchemy 尝试让 DBAPI 驱动程序尽可能多地处理类型安全认证和安全参数字符串连接，其中每个 DBAPI 的具体问题都可以得到适当和安全地处理。

因为SQLAlchemy故意不支持所有字面值的完全字符串操作，因此在特定的调试情况下进行字符串操作的技术包括以下内容。作为示例，我们将使用PostgreSQL :class:`_postgresql.UUID` 数据类型::

    import uuid

    from sqlalchemy import Column
    from sqlalchemy import create_engine
    from sqlalchemy import Integer
    from sqlalchemy import select
    from sqlalchemy.dialects.postgresql import UUID
    from sqlalchemy.orm import declarative_base


    Base = declarative_base()


    class A(Base):
        __tablename__ = "a"

        id = Column(Integer, primary_key=True)
        data = Column(UUID)


    stmt = select(A).where(A.data == uuid.uuid4())

针对用于将列与单个UUID值进行比较的列和语句，呈现包括联接的选项包括：

* 一些DBAPI，例如psycopg2支持辅助函数，如`mogrify（）<https://www.psycopg.org/docs/cursor.html#cursor.mogrify>`_，提供对它们的字面量呈现功能的访问。要使用这些功能，请在不使用``literal_binds``的情况下呈现SQL字符串，并通过 :attr:`.SQLCompiler.params`访问参数::

      e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

      with e.connect() as conn:
          cursor = conn.connection.cursor()
          compiled = stmt.compile(e)

          print(cursor.mogrify(str(compiled), compiled.params))

  上述代码将生成psycopg2的原始字节串：

  .. sourcecode:: sql

      b"SELECT a.id, a.data \nFROM a \nWHERE a.data = 'a511b0fc-76da-4c47-a4b4-716a8189b7ac'::uuid"

* 直接将 :attr:`.SQLCompiler.params` 渲染到语句中，使用
  目标DBAPI的适当位置的  `paramstyle <https://www.python.org/dev/peps/pep-0249/#paramstyle>`_。
  例如，psycopg2 DBAPI使用命名``pyformat``样式。 ``render_postcompile``
  的意义在下一节将被讨论。**警告这是不安全的，不要使用不可信的输入** ::

    e = create_engine("postgresql+psycopg2://")

    # 将使用pyformat样式，例如param%(name)s 用于param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    print(str(compiled) % compiled.params)

  这将产生一个工作不良的字符串，尽管它适用于调试：

  .. sourcecode:: sql

    SELECT a.id, a.data
    FROM a
    WHERE a.data = 9eec1209-50b4-4253-b74b-f82461ed80c1

  另一个例子使用位置的样式,诸如``qmark``，我们还可以使用
  :attr:`.SQLCompiler.positiontup` 集合，同时与 :attr:`.SQLCompiler.params`一起使用，以便编译的语句中能够以其位置为顺序检索参数。::

    import re

    e = create_engine("sqlite+pysqlite://")

    # 会使用qmark样式，也就是? for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    # 按位置的顺序传递参数
    params = (repr(compiled.params[name]) for name in compiled.positiontup)

    # 将经过处理的参数代入查询字符串中
    print(re.sub(r"\?",
                 lambda m: next(params),
                 str(compiled)))

  上部分脚本产生：

  .. sourcecode:: sql

    SELECT a.id, a.data
    FROM a
    WHERE a.data = UUID('1bd70375-db17-4d8c-94f1-fc2ef3aada26')

* 使用 :ref:`sqlalchemy.ext.compiler_toplevel` 扩展，以自定义方式呈现
  :class:`_sql.BindParameter`对象,当用户定义的标志存在时。该标志通过与其他标志一样通过 ``compile_kwargs`` 字典发送::

      from sqlalchemy.ext.compiler import compiles
      from sqlalchemy.sql.expression import BindParameter

      @compiles(BindParameter)
      def _render_literal_bindparam(element, compiler, use_my_literal_recipe=False, **kw):
          if not use_my_literal_recipe:
              # 使用bindparam处理
              return compiler.visit_bindparam(element, **kw)

          # 如果在compiler_kwargs中传递了use_my_literal_recipe，
          # 就将值渲染为直接值
          return repr(element.value)

    e = create_engine("postgresql+psycopg2://")
    print(stmt.compile(e, compile_kwargs={"use_my_literal_recipe": True}))

  上述脚本将给出：

  .. sourcecode:: sql

    SELECT a.id, a.data
    FROM a
    WHERE a.data = UUID('47b154cd-36b2-42ae-9718-888629ab9857')

* 对于内置于模型或语句中的自定义字符串化的特定类型，可以使用
  :class:`_types.TypeDecorator` 类来提供自定义字符串化。使用方法是 :meth:`.TypeDecorator.process_literal_param` 方法 ::

    from sqlalchemy import TypeDecorator

    class UUIDStringify(TypeDecorator):
        impl = UUID

        def process_literal_param(self, value, dialect):
            return repr(value)

  上述数据类型需要明确地在模型中或在语句中使用 :func:`_sql.type_coerce`，例如::

    from sqlalchemy import type_coerce

    stmt = select(A).where(type_coerce(A.data, UUIDStringify) == uuid.uuid4())

    # 显示呈现类型为串联形式，同时输出直接绑定的参数
    print(stmt.compile(e, compile_kwargs={"literal_binds": True}))

联接参数的呈现不像绑定参数那样直截了当，因为绑定参数会由调用的数据库API完成。呈现联接参数的方法自由度不高。需要对特定数据库进行相当多的工作，而且很可能需要写针对每个数据类型的自定义字符串化函数。此外，通篇使用绑定参数更为安全，因为SQLAlchemy会对输入值进行验证并通过正确的机制传递给SQL服务器，从而避免了始终的SQL注入问题。

.. _faq_sql_expression_percent_signs:

生成SQL语句字符串时百分号为什么会被双倍加上？
------------------------------------------------------------------------

很多 :term:`DBAPI` 实现使用 ``pyformat`` 或 ``format`` `paramstyle`，它们在语法上必然涉及百分号。大多数这样做的 DBAPI 都期望百分号用于其他原因的情况下被加倍（例如转义）在使用 SQLAlchemy 传递给底层 DBAPI 的 SQL 语句中，绑定参数的替换方式类似于 Python 字符串插值运算符 ``%`` ，在很多情况下，DBAPI 实际上是直接使用它。在上面，绑定参数的替换看起来像这样：

.. sourcecode:: sql

    SELECT a, b FROM some_table WHERE a = %s AND c = %s AND num %% modulus = 0

当 SQLAlchemy 传递 SQL 语句时，驱动程序代替 SQL 语句中的百分号并使用参数值（在 Python DBAPI 要求的方法中）代替百分号。 在上面的例子中，替换的情况将是：

.. sourcecode:: sql

    SELECT a, b FROM some_table WHERE a = 5 AND c = 10 AND num % modulus = 0

对于使用重新定义了百分号的某些DBAPI，例如PostgreSQL和MySQL，它们具有特定的百分号转义行为：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import table, column
    >>> from sqlalchemy.dialects import postgresql
    >>> t = table("my_table", column("value % one"), column("value % two"))
    >>> print(t.select().compile(dialect=postgresql.dialect()))
    {printsql}SELECT my_table."value %% one", my_table."value %% two"
    FROM my_table

在使用上述方言时，如果需要生成不具有绑定参数符号的非DBAPI语句，则移除百分号的简便方法是使用Python的 ``%`` 运算符直接将其替换为空集：

.. sourcecode:: pycon+sql

    >>> strstmt = str(t.select().compile(dialect=postgresql.dialect()))
    >>> print(strstmt % ())
    {printsql}SELECT my_table."value % one", my_table."value % two"
    FROM my_table

另一个解决方案是在dialect中设置不同的参数样式； 所有 :class:`.Dialect` 实现都接受
``paramstyle`` 参数，它会引起dialect的编译器使用给定的参数格式。 例如，在使用PostgreSQL时，
可以使用非常常见的 `named` 参数样式来设置 Dialect，因此不再需要对百分号进行转义：

.. sourcecode:: pycon+sql

    >>> print(t.select().compile(dialect=postgresql.dialect(paramstyle="named")))
    {printsql}SELECT my_table."value % one", my_table."value % two"
    FROM my_table

.. _faq_sql_expression_op_parenthesis:

我使用op（）生成自定义运算符，但我的圆括号没有正确输出
-----------------------------------------------------------------------------------

:meth:`.Operators.op`方法允许创建自定义的数据库运算符，否则为SQLAlchemy未知：

.. sourcecode:: pycon+sql

    >>> print(column("q").op("->")(column("p")))
    {printsql}q -> p

然而，在将运算符用于复合表达式的右侧时，它不会生成圆括号，如下所示：

.. sourcecode:: pycon+sql

    >>> print((column("q1") + column("q2")).op("->")(column("p")))
    {printsql}q1 + q2 -> p

在上述例子中，我们期望得到 ``(q1 + q2) -> p``。

解决此问题的方法是使用:paramref:`.Operators.op.precedence`参数设置操作数优先级的高值，在100是最大值的情况下，任何SQLAlchemy运算符的运行最高数字是15：

.. sourcecode:: pycon+sql

    >>> print((column("q1") + column("q2")).op("->", precedence=100)(column("p")))
    {printsql}(q1 + q2) -> p

我们也可以使用:meth:`_expression.ColumnElement.self_group`方法来通常强制执行二进制表达式的圆括号（例如，具有左参数/右参数和运算符的表达式）：

.. sourcecode:: pycon+sql

    >>> print((column("q1") + column("q2")).self_group().op("->")(column("p")))
    {printsql}(q1 + q2) -> p

为什么会这样圆括号规则呢？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

很多数据库在存在过多括号或存在于不同于它期望的其他位置的括号时会出现故障，因此 SQLAlchemy 不会基于程序中的括号使用括号，而是使用操作符优先级和已知为关联的操作符确定分组方式。否则，表达式：

    column("a") & column("b") & column("c") & column("d")

会产生：

.. sourcecode:: sql

    (((a AND b) AND c) AND d)

正常情况下没问题，但可能会让人生气（并报告为错误）。在其他情况下，这会产生更难以理解或阅读的表达式，例如：

    column("q", ARRAY(Integer, dimensions=2))[5][6]

将产生：

.. sourcecode:: sql

    ((q[5])[6])

有时会出现``"(x) = 7"``等情况，数据库真的不喜欢这种写法。因此，括号不会简单地排列，而是使用运算符的优先级和运算符的综合性来确定分组方式。

对于 :meth:`.Operators.op`，优先级的默认值为零。