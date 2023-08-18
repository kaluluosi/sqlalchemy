.. admonition:: 我们已经迁移！

    这个页面曾经是SQLAlchemy 1.x教程的旧址。从2.0版本开始，SQLAlchemy采用了一种经过修订的工作方式，并提供了全新的教程，在最新的使用模式中集成了核心和ORM。请见   :ref:`unified_tutorial` 。


Introduction
------------

The SQLAlchemy SQL Expression Language presents a system of representing
structured data in Python. It provides a simple way to express complex queries
in a way that is familiar to SQL users, leveraging all the benefits of the
Python programming language like expression syntax, namedtuple & object
relational mapping, and rich data types.

.. note::

    The expressions in the SQL Expression Language are constructed in Python
    using regular Python statements, not SQLAlchemy-specific strings.  All the
    benefits of Python code - including error handling, code completion,
    refactoring tools, and more - are available when constructing SQL
    expressions using the SQL Expression Language.


The Basics
----------

SQLAlchemy's SQL Expression Language makes use of Python's powerful expression
language to make writing SQL statements easy.

Consider a users table with columns id, name, age, and created_date.

Here's a simple select statement::

    from sqlalchemy.sql import select

    users = Table('users', metadata,
            Column('id', Integer, primary_key=True),
            Column('name', String),
            Column('age', Integer),
            Column('created_date', DateTime),
            )

    # select * from users
    select_stmt = select([users])

This statement will build a SQL expression that will select all columns from a
users table. The   :func:`.select`  function takes a list of columns or expressions
to return from the SELECT statement.

In this example, the `Table` object is defined with the ``metadata`` object as
a parent object, which represents all the database objects in SQLAlchemy. Once
we have the table, we can build a ``select()`` statement that returns all
columns of that table by passing the ``Table`` instance to the select function.

Expressions can be combined using Python operators to provide more complex
queries. For example, we can select only the ``id`` and ``name`` columns of
users under 30 years old::

    # select name, id from users where age < 30
    select_stmt = select([users.c.name, users.c.id]).\
                    where(users.c.age < 30)

In this case, we use the ``where()`` method to filter only rows whose ``age``
column is less than 30.

.. note::

    It is always necessary to specify the ``.c`` attribute before using the
    column object.

The result of the select statement is a SQLAlchemy select object, which can be
executed using an engine's  :meth:`.execute`  method. An engine is SQLAlchemy's
core interface for executing SQL statements.