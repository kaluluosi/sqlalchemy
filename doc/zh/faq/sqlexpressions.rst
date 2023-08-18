SQL表达式
===============

.. contents::
    :local:
     :class: faq
    :backlinks: none

.. _faq_sql_expression_string:

如何将SQL表达式呈现为字符串，可能带有内联的绑定参数？
------------------------------------------------------------

在大多数简单情况下，将SQLAlchemy核心语句对象或ORM的“stringification”  :class:`_query.Query` 对象或
表达式片段和其他任何表达式片段呈现为字符串的方法是非常简单的，只需使用``str()``内置函数即可，
例如当使用它与 ``print`` 函数（请注意，如果不显示调用 `str()`，Python的`print`函数也会自动调用它）：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import table, column, select
    >>> t = table("my_table", column("x"))
    >>> statement = select(t)
    >>> print(str(statement))
    {printsql}SELECT my_table.x
    FROM my_table

对于ORM的  :class:`_query.Query` ，  :func:` _expression.insert`等）
以及任何表达式片段，都可以调用``str()``内置函数或等效函数：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import column
    >>> print(column("x") == "some value")
    {printsql}x = :x_1

特定数据库的字符串呈现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当要将要呈现为字符串的语句包含具有特定于数据库的字符串格式的元素时，
或者当包含元素仅在某一种类型的数据库中才可用时，就会产生问题。
在这些情况下，我们可能会得到一个字符串化的语句，该语句不符合我们目标的数据库的正确语法，
或者该操作可能会引发  :class:`.UnsupportedCompilationError` 异常。在这些情况下，我们需要使用
  :meth:`_expression.ClauseElement.compile`  方法将语句字符串化，同时传递一个代表目标数据库的
  :class:`_engine.Engine` .Dialect` 对象。例如，如果我们有一个MySQL数据库引擎，
我们可以使用以下方法将语句字符串化为MySQL方言::

    from sqlalchemy import create_engine

    engine = create_engine("mysql+pymysql://scott:tiger@localhost/test")
    print(statement.compile(engine))

可以直接使用 :class:`.Dialect` 对象，如下例所示，我们使用PostgreSQL方言::

    from sqlalchemy.dialects import postgresql

    print(statement.compile(dialect=postgresql.dialect()))

请注意，任何方言都可以使用   :func:`_sa.create_engine`  本身来组装，
使用虚拟URL，然后访问 :attr:`_engine.Engine.dialect` 属性，例如，
如果我们想要一个psycopg2的方言对象::

    e = create_engine("postgresql+psycopg2://")
    psycopg2_dialect = e.dialect

当给定ORM的  :class:`~.orm.query.Query` ，要访问  :meth:` _expression.ClauseElement.compile`  
方法，我们只需要首先访问 :attr:`~.orm.query.Query.statement` 访问器：

    statement = query.statement
    print(statement.compile(someengine))

内联绑定参数呈现
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

.. warning:: **决不要**在不可信的输入（例如来自Web表单或其他用户输入应用程序的字符串内容）中使用这些技术。
SQLAlchemy的将Python值转换为直接SQL字符串值的工具对来自不可信输入的数据 **不安全且不验证传递的数据类型**。在与关系数据库
编程调用非DDL SQL语句时，始终使用绑定参数。

上面的形式将呈现SQL语句传递给Python :term:`DBAPI` 时，包括未呈现内联的绑定参数。
SQLAlchemy通常不会将绑定参数字符串化，这由Python DBAPI适当处理，更不用说绕过绑定参数可能是现代Web应用程序中最常被利用的安全漏洞。
在检查DDL发射的情况下，SQLAlchemy具有以下所述的这种字符串化功能的有限能力。为了访问此功能，可以使用传递给``compile_kwargs``的
``literal_binds``标志：

    from sqlalchemy.sql import table, column, select

    t = table("t", column("x"))

    s = select(t).where(t.c.x == 5)

    # **不要在受信任的输入之外使用**！
    print(s.compile(compile_kwargs={"literal_binds": True}))

    # 为特定方言呈现
    print(s.compile(dialect=dialect, compile_kwargs={"literal_binds": True}))

    # 或者，如果您有一个Engine，请作为第一个参数传递
    print(s.compile(some_engine, compile_kwargs={"literal_binds": True}))

此功能主要用于记录或调试的目的，获取查询的原始SQL字符串可能证明有用。

上述方法的注意事项是它仅支持基本类型，例如整数和字符串，而且如果直接使用   :func:`.bindparam`  而没有预设值，也无法将其字符串化。
无条件地字符串化所有参数的方法如下所述。

.. tip::

   SQLAlchemy不支持完全字符串序列化所有数据类型的原因归结为以下三点：

   1. 当正常使用DBAPI时，该DBAPI已经支持此功能。SQLAlchemy项目不能负责为每种后端复制所有数据类型的重复性工作，并且这是多余的工作，还会
      带来显着的测试和持续支持开销。
   2. 呈现特定数据库的绑定的完全字符串化暗示了一种用法，即实际将这些完全字符串化的语句传递到数据库进行执行。这是不必要和不安全的，
      SQLAlchemy不希望以任何方式鼓励此使用方式。
   3. 文字值呈现领域最可能发现安全问题。SQLAlchemy尽量使安全的参数字符串化领域尽可能成为DBAPI驱动程序的问题，其中每个DBAPI的具体信息可以得到适当和安全的处理。

由于意图不支持完全字符串化文字值，因此在特定调试场景下执行此操作的技术包括以下内容。以PostgreSQL   :class:`_postgresql.UUID`  数据类型为例：

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

在上面的模型和语句中，将列与单个UUID值进行比较，包含内联值的SQL语句呈现选项包括：

* 一些DBAPI（例如psycopg2）支持像 `mogrify()<https://www.psycopg.org/docs/cursor.html#cursor.mogrify>`_ 这样的辅助函数，
  提供了在它们的文字网格填充中使用的字面量渲染功能。要使用此类功能，请渲染SQL字符串而不使用
  ``literal_binds``，并通过 :attr:`.SQLCompiler.params` 访问参数本身::

      e = create_engine("postgresql+psycopg2://scott:tiger@localhost/test")

      with e.connect() as conn:
          cursor = conn.connection.cursor()
          compiled = stmt.compile(e)

          print(cursor.mogrify(str(compiled), compiled.params))

  上述代码将产生psycopg2的原始字节串：

  .. sourcecode:: sql

      b"SELECT a.id, a.data \nFROM a \nWHERE a.data = 'a511b0fc-76da-4c47-a4b4-716a8189b7ac'::uuid"

* Render the  :attr:`.SQLCompiler.params`  直接插入语句中，使用目标DBAPI的与` paramstyle <https://www.python.org/dev/peps/pep-0249/#paramstyle>`_相对应的
  样式。例如，psycopg2 DBAPI使用命名的 ``pyformat`` 样式。``render_postcompile``的含义将在下一部分中讨论。**警告**此时**不安全，不要使用不可信、无限制的输入**：

    e = create_engine("postgresql+psycopg2://")

    # Will use pyformat style, i.e. %(paramname)s for param.
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    print(str(compiled) % compiled.params)

  这将生成一个不工作的字符串，尽管适合调试：

  .. sourcecode:: sql

    SELECT a.id, a.data
    FROM a
    WHERE a.data = 9eec1209-50b4-4253-b74b-f82461ed80c1

  另一个例子是使用qmark之类的位置参数样式，我们可以通过还使用  :attr:`.SQLCompiler.lineterminator`  、  :attr:` .SQLCompiler.positional`  ，以便组合渲染线上参数到一个字符串（线上参数是  :attr:`.SQLCompiler.positiontup`  集合， 处理时默认是呈现为未绑定参数，这样它们在生成的SQL字符串中将不会得到引用）。示例适用于使用SQLite的情况：

    import re

    e = create_engine("sqlite+pysqlite://")

    # 接受qmark style, i.e. ? for param
    compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})

    # 确定语句的绑定参数的项的位置
    params = (repr(compiled.params[name]) for name in compiled.positiontup)

    print(re.sub(r"\?", lambda m: next(params), str(compiled)))

  上述片段打印：

  .. sourcecode:: sql

    SELECT a.id, a.data
    FROM a
    WHERE a.data = UUID('1bd70375-db17-4d8c-94f1-fc2ef3aada26')

* 使用   :ref:`sqlalchemy.ext.compiler_toplevel`  扩展在具有自定义计算时呈现   :class:` _sql.BindParameter`  对象的方式在用户定义的标志存在时，来自编译器的自定义方式。该标志通过类似于其他标志的编译器_kwargs字典发送：

    from sqlalchemy.ext.compiler import compiles
    from sqlalchemy.sql.expression import BindParameter

    @compiles(BindParameter)
    def _render_literal_bindparam(element, compiler, use_my_literal_recipe=False, **kw):
        if not use_my_literal_recipe:
            # 使用正常的bindparam处理
            return compiler.visit_bindparam(element, **kw)

        # 如果在compiler_kwargs中传递了use_my_literal_recipe，将值直接渲染出来
        return repr(element.value)

    e = create_engine("postgresql+psycopg2://")
    print(stmt.compile(e, compile_kwargs={"use_my_literal_recipe": True}))

上面的配方将打印：

  .. sourcecode:: sql

    SELECT a.id, a.data
    FROM a
    WHERE a.data = UUID('47b154cd-36b2-42ae-9718-888629ab9857')

* 对于内建到模型或语句中的类型特定字符串化，可以使用   :class:`_types.TypeDecorator`  类提供  :meth:` .TypeDecorator.process_literal_param`  方法自定义任何数据类型的字符串化方式。

  示例数据类型需要在模型中显式使用，或者在语句中使用   :func:`_sql.type_coerce`  本地模拟，例如：

    from sqlalchemy import type_coerce

    stmt = select(A).where(type_coerce(A.data, UUIDStringify) == uuid.uuid4())

    print(stmt.compile(e, compile_kwargs={"literal_binds": True}))

  会再次打印同样的表达式：

  .. sourcecode:: sql

    SELECT a.id, a.data
    FROM a
    WHERE a.data = UUID('47b154cd-36b2-42ae-9718-888629ab9857')

将 "POSTCOMPILE" 参数呈现为绑定参数
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

SQLAlchemy包括一种变体的绑定参数，称为  :paramref:`_sql.BindParameter.expanding` ，它是一种“延迟评估”的参数，当SQL构造被编译时，它呈现在中间状态，
然后在语句执行时进一步处理已知的实际值。类似这种结构的"id in [...]"使用可以在SQL字符串在实际列表值无关的情况下安全地缓存：

  >>> stmt = select(A).where(A.id.in_[1, 2, 3])

要将IN子句呈现为实际的绑定参数符号，请使用 ``render_postcompile=True`` 标志与  :meth:`_sql.ClauseElement.compile` ：

.. sourcecode:: pycon+sql

  >>> e = create_engine("postgresql+psycopg2://")
  >>> print(stmt.compile(e, compile_kwargs={"render_postcompile": True}))
  {printsql}SELECT a.id, a.data
  FROM a
  WHERE a.id IN (%(id_1_1)s, %(id_1_2)s, %(id_1_3)s)

在前面呈现绑定参数的形式的示例中，``literal_binds``标志（描述呈现绑定参数的形式）会自动将 ``render_postcompile`` 标志设置为True，这样对于只有简单的整数/字符串数的语句，可以直接呈现它们：

.. sourcecode:: pycon+sql

  # render_postcompile是文字绑定的意味着
  >>> print(stmt.compile(e, compile_kwargs={"literal_binds": True}))
  {printsql}SELECT a.id, a.data
  FROM a
  WHERE a.id IN (1, 2, 3)

  :attr:`.SQLCompiler.params`   和  :attr:` .SQLCompiler.positiontup`  也可以与 ``render_postcompile``兼容，因此在这里呈现内联绑定参数的先前配方，就以同样的方式起作用，例如在SQLite的位置形式：

.. sourcecode:: pycon+sql

  >>> u1, u2, u3 = uuid.uuid4(), uuid.uuid4(), uuid.uuid4()
  >>> stmt = select(A).where(A.data.in_([u1, u2, u3]))

  >>> import re
  >>> e = create_engine("sqlite+pysqlite://")
  >>> compiled = stmt.compile(e, compile_kwargs={"render_postcompile": True})
  >>> params = (repr(compiled.params[name]) for name in compiled.positiontup)
  >>> print(re.sub(r"\?", lambda m: next(params), str(compiled)))
  {printsql}SELECT a.id, a.data
  FROM a
  WHERE a.data IN (UUID('aa1944d6-9a5a-45d5-b8da-0ba1ef0a4f38'), UUID('a81920e6-15e2-4392-8a3c-d775ffa9ccd2'), UUID('b5574cdb-ff9b-49a3-be52-dbc89f087bfa'))

.. warning::

    记住，上述所有字面量值字符串化的代码配方，是在**仅用于**以下情况：

    1. 使用了 **仅供调试** 的目的

    2. 字符串 **不会被传递到生产数据库**

    3. 只用于 **本地、信任的输入**

    上述呈现字面值的配方为**任何情况**都不安全，绝不应在生产数据库上使用。

.. _faq_sql_expression_percent_signs:

为什么字符串化SQL语句时百分号会被双倍？
--------------------------------------------------------

许多DBAPI实现使用 ``pyformat`` 或 ``format`` `paramstyle <https://www.python.org/dev/peps/pep-0249/#paramstyle>`_ ，这些实现必然涉及到其语法中的百分号。
大多数在工作中使用此功能的DBAPI期望在使用有其他含义的百分号时将百分号加倍（即转义），例如：

.. sourcecode:: sql

    SELECT a, b FROM some_table WHERE a = %s AND c = %s AND num %% modulus = 0

当SQLAlchemy将语句传递给底层DBAPI时，绑定参数的替换方式与Python字符串插值操作符``%``相同，在许多情况下，实际上DBAPI直接使用此运算符。
在前面的示例中，绑定参数的替换如下所示::

.. sourcecode:: sql

    SELECT a, b FROM some_table WHERE a = 5 AND c = 10 AND num % modulus = 0

如PostgreSQL（默认DBAPI是psycopg2）和MySQL（默认DBAPI是mysqlclient）这样的数据库的默认编译器遵循此百分号转义行为：

.. sourcecode:: pycon+sql

    >>> from sqlalchemy import table, column
    >>> from sqlalchemy.dialects import postgresql
    >>> t = table("my_table", column("value % one"), column("value % two"))
    >>> print(t.select().compile(dialect=postgresql.dialect()))
    {printsql}SELECT my_table."value %% one", my_table."value %% two"
    FROM my_table

在使用这种方言时，如果希望获取不包含绑定参数符号的非-DBAPI语句，其中一个快速的方法是使用Python的 ``%`` 操作符将其替换为空参数集：

.. sourcecode:: pycon+sql

    >>> strstmt = str(t.select().compile(dialect=postgresql.dialect()))
    >>> print(strstmt % ())
    {printsql}SELECT my_table."value % one", my_table."value % two"
    FROM my_table

另一个选择是在正在编译的方言中设置不同的参数样式；所有   :class:`.Dialect`  实现都支持一个参数
``paramstyle`` ，它将导致该 dialect 的编译器使用给定的参数样式。例如，设置PostgreSQL dialet中使用广泛的``named``参数样式（$paramname）的dialect：

.. sourcecode:: pycon+sql

    >>> print(t.select().compile(dialect=postgresql.dialect(paramstyle="named")))
    {printsql}SELECT my_table."value % one", my_table."value % two"
    FROM my_table


.. _faq_sql_expression_op_parenthesis:

我正在使用 op() 生成自定义操作符，但是我的括号没有正确显示
-------------------------------------------------------

  :meth:`.Operators.op`   方法允许创建 SQLAlchemy 不知道的自定义数据库操作符：


.. sourcecode:: pycon+sql

    >>> print(column("q").op("->")(column("p")))
    {printsql}q -> p

然而，当在复合表达式的右侧使用它时，它不会产生我们期望的括号：

.. sourcecode:: pycon+sql

    >>> print((column("q1") + column("q2")).op("->")(column("p")))
    {printsql}q1 + q2 -> p

在上面的例子中，我们可能希望得到 ``(q1 + q2) -> p``。

这种情况的解决方案是，使用  :paramref:`.Operators.op.precedence`  参数，将操作符的优先级设置为较高的数字，其中100是最大值，
SQLAlchemy当前使用任何操作符中的最高数字为15：

.. sourcecode:: pycon+sql

    >>> print((column("q1") + column("q2")).op("->", precedence=100)(column("p")))
    {printsql}(q1 + q2) -> p

通常，我们也可以使用  :meth:`_expression.ColumnElement.self_group`  方法强制将二进制表达式(例如拥有左/右操作数和运算符的表达式)括在括号内:

.. sourcecode:: pycon+sql

    >>> print((column("q1") + column("q2")).self_group().op("->")(column("p")))
    {printsql}(q1 + q2) -> p

为什么括号规则是这样的？
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

许多数据库在存在过多括号或括号位于不寻常的位置时都会失败，
因此 SQLAlchemy 不基于分组生成括号，它使用运算符优先级和如果运算符已知是可结合的，
也会生成最少量的括号。 否则，在表达式如下的情况下，留给数据库的表达式将会更易混淆或者至少会影响可读性：

.. sourcecode:: pycon+sql

    column("a") & column("b") & column("c") & column("d")

在其他情况下，会导致更有可能混淆数据库或至少使读取困难，例如::

    column("q", ARRAY(Integer, dimensions=2))[5][6]

会给生成的SQL语句加更多的括号，如(::

    ((q[5])[6])

还有一些边缘情况，其中我们得到像“（x）=7”这样的语句，数据库也不会真正喜欢。因此，
括号不使用组合深度，而是使用运算符优先级和关联性来确定分组关系。

对于  :meth:`.Operators.op` ，优先级的值默认为零。

如果我们将  :paramref:`.Operators.op.precedence`  的值默认为100，即最高值，例如：

.. sourcecode:: pycon+sql

    >>> print((column("q") - column("y")).op("+", precedence=100)(column("z")))
    {printsql}(q - y) + z{stop}
    >>> print((column("q") - column("y")).op("+")(column("z")))
    {printsql}q - y + z{stop}

但这两个不会：

.. sourcecode:: pycon+sql

    >>> print(column("q") - column("y").op("+", precedence=100)(column("z")))
    {printsql}q - y + z{stop}
    >>> print(column("q") - column("y").op("+")(column("z")))
    {printsql}q - (y + z){stop}

到目前为止，似乎不确定是否有一种方法格外括号化自动生成的操作符来自动处理通用操作符的情况，在没有给出优先级的情况下，
同时对于在其他情况下仍重要的现有运算符也可能需要更具体的括号罗辑。这种变化可能可以在某个时候完成，
但是在当前，保持括号化规则的内部一致性似乎是更安全的方法。