特殊关系的持久化模式
======================

.. _post_update:

自我指向/相互依赖行
--------------------

这是一个非常特定的情况，如果要正确填充行（反之亦然），则必须在relationship()中执行INSERT和第二个UPDATE（以及UPDATE和DELETE以删除而不违反外键约束）。这两种用例是：

* 表包含对自身的外键，单个行将具有指向其自身主键的外键值。
* 两个表都包含引用另一个表的外键，每个表中的行都引用另一个表。

例如：

.. sourcecode:: text

              user
    ---------------------------------
    user_id    name   related_user_id
       1       'ed'          1

或：

.. sourcecode:: text

                 widget                                                  entry
    -------------------------------------------             ---------------------------------
    widget_id     name        favorite_entry_id             entry_id      name      widget_id
       1       'somewidget'          5                         5       'someentry'     1

在第一种情况下，一行指向自身。 从技术上讲，使用诸如PostgreSQL或Oracle的序列的数据库可以使用先前生成的值一次性插入行，但依赖于自动增量样式主键标识符的数据库无法。  :func:`~sqlalchemy.orm.relationship` ~sqlalchemy.orm.relationship` 需要使用两个语句。

在第二种情况下，“widget”行必须在任何引用的“entry”行之前插入，但是然后该“widget”行的“favorite_entry_id”列直到生成“entry”行后才能设置。 在这种情况下，通常不可能使用仅使用两个INSERT语句插入“widget”和“entry”行; 必须执行UPDATE以保持外键约束满足。 例外情况是如果将外键配置为“延迟到提交”（某些数据库支持的功能）并手动填充标识符（再次可以绕过  :func:`~sqlalchemy.orm.relationship` ）。

为了启用使用补充UPDATE语句，我们使用  :paramref:`_orm.relationship.post_update`  选项of   :func:` _orm.relationship` 。 这指示必须在INSERT两个行之后使用UPDATE语句创建它们之间的链接，并且需要在DELETE发出之前通过UPDATE解除它们之间的关联。 应在其中一个关系上放置该标志，最好是多对一侧。 下面我们演示完整的示例，包括两个  :class:`_schema.ForeignKey`  constructs::

    from sqlalchemy import Integer, ForeignKey
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Entry(Base):
        __tablename__ = "entry"
        entry_id = mapped_column(Integer, primary_key=True)
        widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
        name = mapped_column(String(50))


    class Widget(Base):
        __tablename__ = "widget"

        widget_id = mapped_column(Integer, primary_key=True)
        favorite_entry_id = mapped_column(
            Integer, ForeignKey("entry.entry_id", name="fk_favorite_entry")
        )
        name = mapped_column(String(50))

        entries = relationship(Entry, primaryjoin=widget_id == Entry.widget_id)
        favorite_entry = relationship(
            Entry, primaryjoin=favorite_entry_id == Entry.entry_id, post_update=True
        )

当针对上述配置的结构进行刷新时，“widget”行将被插入（减少“favorite_entry_id”值），然后所有“entry”行都将被INSERT引用父“widget”行，然后更新语句将填充“widget”表的“favorite_entry_id”列（当前每次一行）：

.. sourcecode:: pycon+sql

    >>> w1 = Widget(name="somewidget")
    >>> e1 = Entry(name="someentry")
    >>> w1.favorite_entry = e1
    >>> w1.entries = [e1]
    >>> session.add_all([w1, e1])
    >>> session.commit()
    {execsql}BEGIN (implicit)
    INSERT INTO widget (favorite_entry_id, name) VALUES (?, ?)
    (None, 'somewidget')
    INSERT INTO entry (widget_id, name) VALUES (?, ?)
    (1, 'someentry')
    UPDATE widget SET favorite_entry_id=? WHERE widget.widget_id = ?
    (1, 1)
    COMMIT

我们可以指定的其他配置是在``Widget``上提供更全面的外键约束，以确保"favorite_entry_id"指向一个也引用此“Widget”的“Entry”。 我们可以使用复合外键来做到这一点，如下所示：

    from sqlalchemy import (
        Integer,
        ForeignKey,
        String,
        UniqueConstraint,
        ForeignKeyConstraint,
    )
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Entry(Base):
        __tablename__ = "entry"
        entry_id = mapped_column(Integer, primary_key=True)
        widget_id = mapped_column(Integer, ForeignKey("widget.widget_id"))
        name = mapped_column(String(50))
        __table_args__ = (UniqueConstraint("entry_id", "widget_id"),)


    class Widget(Base):
        __tablename__ = "widget"

        widget_id = mapped_column(Integer, autoincrement="ignore_fk", primary_key=True)
        favorite_entry_id = mapped_column(Integer)

        name = mapped_column(String(50))

        __table_args__ = (
            ForeignKeyConstraint(
                ["widget_id", "favorite_entry_id"],
                ["entry.widget_id", "entry.entry_id"],
                name="fk_favorite_entry",
            ),
        )

        entries = relationship(
            Entry, primaryjoin=widget_id == Entry.widget_id, foreign_keys=Entry.widget_id
        )
        favorite_entry = relationship(
            Entry,
            primaryjoin=favorite_entry_id == Entry.entry_id,
            foreign_keys=favorite_entry_id,
            post_update=True,
        )

上述映射具有连接“widget_id”和“favorite_entry_id”列的复合  :class:`_schema.ForeignKeyConstraint` 。 为确保` `Widget.widget_id``保持为“自增”列，我们在  :class:`_schema.Column`  设置为` `"ignore_fk"``，并且在每个 :func:`_orm.relationship` 上我们必须将这些列限制为只考虑为了连接和交叉填充。

.. _passive_updates:

可变主键/更新级联
---------------------

当实体的主键发生更改时，引用主键的相关项也必须进行更新。 对于强制引用完整性的数据库，最佳策略是使用数据库的ON UPDATE CASCADE功能，以便将主键更改传播到引用的外键 - 值不能在任何瞬间不同，除非将约束标记为“可延迟”，即在事务完成之前不会实施。

**强烈建议**使用数据库的``ON UPDATE CASCADE``功能的应用程序使用可变值的自然主键。下面是一个演示此类自然主键的示例映射：

    class User(Base):
        __tablename__ = "user"
        __table_args__ = {"mysql_engine": "InnoDB"}

        username = mapped_column(String(50), primary_key=True)
        fullname = mapped_column(String(100))

        addresses = relationship("Address")


    class Address(Base):
        __tablename__ = "address"
        __table_args__ = {"mysql_engine": "InnoDB"}

        email = mapped_column(String(50), primary_key=True)
        username = mapped_column(
            String(50), ForeignKey("user.username", onupdate="cascade")
        )

在上面的示例中，我们说明了  :class:`_schema.ForeignKey` ` onupdate="cascade"``，并且我们还说明了``mysql_engine='InnoDB'``设置，这样，在使用MySQL后端时，就会使用支持引用完整性的``InnoDB``引擎。 在SQLite上使用引用完整性时，应使用描述在：ref：'sqlite_foreign_keys'中的配置。

.. seealso::

      :ref:`passive_deletes` -支持关系ON DELETE CASCADE的功能

     :paramref:`.orm.mapper.passive_updates` -   :class:` _orm.Mapper`  上类似的功能


模拟没有外键支持的受限ON UPDATE CASCADE
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
在使用不支持引用完整性的数据库并播放具有可变值的自然主键的情况下，SQLAlchemy提供了一种功能，以允许仅在对立即引用到主键列已更改值的外键列发出UPDATE语句的情况下，将主键值的传播传播到已引用的外键，到**有限的范围**。
不支持引用完整性功能的主要平台是在使用“ MyISAM”存储引擎时的MySQL以及不使用“ PRAGMA foreign_keys = ON”命令的SQLite。Oracle数据库也不支持``ON UPDATE CASCADE``，但是因为它仍强制执行引用完整性，需将约束标记为可延迟以便SQLAlchemy能够发出UPDATE语句。

将  :paramref:`_orm.relationship.passive_updates`   标志设置为` `False``即可启用该功能，最好是在一个一对多或多对多 :func:`_orm.relationship` 上使用。当“更新”不再是“被动”的时候，这表明SQLAlchemy将针对集合中预处理并发出UPDATE语句对与具有变化的主键值相应的父对象相关联的对象。这也意味着，如果尚未本地存在，则会完全将集合加载到内存中。

我们使用``passive_updates=False``的先前映射如下所示：

    class User(Base):
        __tablename__ = "user"

        username = mapped_column(String(50), primary_key=True)
        fullname = mapped_column(String(100))

        # 如果数据库不实现ON UPDATE CASCADE，则只需要passive_updates = False
        # 注意，在Oracle上需要将所有约束标记为可延迟
        addresses = relationship("Address", passive_updates=False)


    class Address(Base):
        __tablename__ = "address"

        email = mapped_column(String(50), primary_key=True)
        username = mapped_column(String(50), ForeignKey("user.username"))

`passive_updates=False`的主要局限性包括：

它的执行效率远远低于直接使用数据库ON UPDATE CASCADE，因为它需要使用SELECT将受影响的集合完全预加载，并且必须针对那些值发出UPDATE语句，它将尝试以“批处理”的方式运行，但仍然以DBAPI级别逐行运行。

该功能不能“级联”超过一级。 也就是说，如果X映射具有引用到映射Y的主键的外键，但是然后映射Y的主键本身是一个外键，指向映射Z，则`passive_updates=False`不能从`Z`级联更改主键值到`X`级别。

在关系多对一侧上仅配置``passive_updates=False``不会有完全效果，因为工作单元仅在当前标识映射中搜索可能引用具有变异主键的对象，而不是遍及整个数据库。