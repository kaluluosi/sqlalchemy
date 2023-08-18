特殊关系持久化模式
=========================

.. _post_update:

自身引用的行/互相依赖的行
-----------------------------------------------

这是一种非常特殊的情况，需要 :func:`~sqlalchemy.orm.relationship` 执行一个 INSERT 和一个UPDATE 以正确地填充一行（反之，需要一个 UPDATE 和一个 DELETE，以便在不违反外键约束的情况下进行删除）。这两种用例是：

* 表格包含外键指向自身，并且单个行将有一个外键值指向其自己的主键。
* 两个表格各包含一个外键，引用另一个表格，在每个表格中都有引用另一个表格的行。

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

在第一种情况下，一行指向自身。从技术上讲，使用序列的数据库（如 PostgreSQL 或 Oracle）可以一次 INSERT 行使用先前生成的值，但是依赖 Autoincrement-style 主键标识符的数据库则不行。:func:`~sqlalchemy.orm.relationship` 总是假定“父/子”模型的行填充过程在 flush 期间进行，因此，除非直接填充主键/外键列，否则 :func:`~sqlalchemy.orm.relationship` 需要使用两个语句。

在第二种情况下，“widget”行必须在任何引用“entry”行之前插入，但是然后不能设置该“widget”行的“favorite_entry_id”列，直到生成了“entry”行。在这种情况下，通常不可能只使用两个 INSERT 语句插入“widget”和“entry”行；必须执行 UPDATE 以保持外键约束满足条件。除非将外键配置为“延迟到提交”（某些数据库支持的功能），并且如果标识符是手动填充的（再次基本上绕过了 :func:`~sqlalchemy.orm.relationship`）。

为了使用补充 UPDATE 语句，我们使用 :paramref:`_orm.relationship.post_update` 选项，这个选项是 :func:`_orm.relationship` 中的。这指定两个行之间的关系应在两个行插入后使用 UPDATE 语句创建；它还导致两个行在 DELETE 发出之前通过 UPDATE 取消关联。标记应放在关系中的 *一个*，最好是多对一侧。下面我们说明一个完整的示例，包括两个 :class:`_schema.ForeignKey` 结构：

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

当使用上面的配置让在被刷新时，"widget" 行将被 INSERT，但是 "favorite_entry_id" 值除外，然后所有 "entry" 行都将参照父 "widget" 行插入，然后将使用 UPDATE 语句填充 "widget" 表的 "favorite_entry_id" 列（这是目前的一行一行）：

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

我们可以指定一个更细致的外键约束在“Widget”上，这样就可以保证“favorite_entry_id”指的是一个也引用了这个“Widget”的“Entry”。我们可以使用一个复合的外键，如下所示：

    from sqlalchemy import Integer, ForeignKey
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship
    from sqlalchemy import String, UniqueConstraint, ForeignKeyConstraint

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

上述映射具有跨 "widget_id" 和 "favorite_entry_id" 列拼合的复合 :class:`_schema.ForeignKeyConstraint`。为了确保 "Widget.widget_id" 仍是一个“自增”列，我们在 :class:`_schema.Column` 上指定了 :paramref:`_schema.Column.autoincrement` 的值为“ignore_fk”，并且还必须在每个 :func:`_orm.relationship` 上限制那些作为连接和交叉填充中的外键的列。

.. _passive_updates:

可变主键/更新级联
--------------------------

当实体的主键更改时，引用主键的相关项也必须进行更新。对于强制引用完整性的数据库，最佳策略是使用数据库的 ON UPDATE CASCADE 功能，以便将主键更改传播到引用的外键 - 除非将约束标记为“可延迟”，否则值不能短暂地不同步。

如果一个应用程序使用可变值的自然主键，则强烈建议使用数据库的 ``ON UPDATE CASCADE`` 功能。下面示例演示：

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

上面，我们在 :class:`_schema.ForeignKey` 上说明了 ``onupdate="cascade"``，并且还说明了 ``mysql_engine='InnoDB'`` 设置，在 MySQL 后端上，这确保了支持执行引用完整性的引擎“ InnoDB”。在使用 SQLite 时，应启用引用完整性，使用 :ref:`sqlite_foreign_keys` 描述的配置。


.. seealso::

    :see:`passive_deletes` - 支持关系中的 ON DELETE CASCADE

    :paramref:`.orm.mapper.passive_updates` - :class:`sqlalchemy.orm.Mapper` 中的类似特性


在没有外键支持的情况下模拟受限的 ON UPDATE CASCADE
----------------------------------------------------------

在使用不支持引用完整性的数据库，并且使用可变值的自然主键时，SQLAlchemy 提供一种特性，以允许将主键值传播到已引用的外键，但是只有在某种程度上。限制，通过针对外键列发出 UPDATE 语句，立即引用已更改主键列的主键列，仅在无法使用 PRAGMA foreign_keys=ON 的情况下才启用该功能，这些主要平台是使用 MyISAM 存储引擎时的 MySQL，以及没有使用该引用完整性的 SQLite。Oracle 数据库也不支持 ``ON UPDATE CASCADE``，但因为它仍然执行引用完整性，需要将约束标记为可延迟，以便 SQLAlchemy 可以发出 UPDATE 语句。

通过将 :paramref:`_orm.relationship.passive_updates` 标志设置为 ``False`` 以启用该功能，最好是在一对多或多对多的 :func:`_orm.relationship` 上。当“更新”不再是“被动的”时，这表示 SQLAlchemy 将针对具有变异性主键值的更改的主父对象所引用的集合中的对象单独发出 UPDATE 语句。这也意味着，如果尚未本地存在，集合将完全加载到内存中。

使用 ``passive_updates=False`` 的先前映射如下所示：

    class User(Base):
        __tablename__ = "user"

        username = mapped_column(String(50), primary_key=True)
        fullname = mapped_column(String(100))

        addresses = relationship("Address", passive_updates=False)


    class Address(Base):
        __tablename__ = "address"

        email = mapped_column(String(50), primary_key=True)
        username = mapped_column(String(50), ForeignKey("user.username"))

``passive_updates=False`` 的主要限制包括：

* 它的性能比直接数据库 ON UPDATE CASCADE 差得多，因为它需要使用 SELECT 完全预加载受影响的集合，并且还必须对这些值发出 UPDATE 语句，它将尝试以“批处理”的方式运行，但仍然在 DBAPI 级别上以逐行方式运行。

* 该功能不能“级联”超过一级。也就是说，如果映射 X 有一个外键引用了映射 Y 的主键，但然后映射 Y 的主键本身是指向映射 Z 的外键，那么 ``passive_updates=False`` 无法将主键值的更改从 ``Z`` 传播到 ``X``。

* 在关系中仅在多对一侧上配置“passive_updates=False”将无法产生全面的效果，因为单元操作仅在当前身份映射中搜索可能引用具有可变主键的对象，而不是在整个数据库中搜索。


因为几乎所有数据库现在都支持 ``ON UPDATE CASCADE``，因此强烈建议在使用自然和可变主键值时使用传统的 ``ON UPDATE CASCADE`` 支持。