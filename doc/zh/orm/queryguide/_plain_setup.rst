:rst引用标记前后需加空格隔开:

======================================
ORM 查询指南的设置：SELECT
======================================

本页说明了   :ref:`queryguide_toplevel`  中的  :doc:` select`  文档使用的映射和样例数据。

..  sourcecode:: python

    >>> from typing import List
    >>> from typing import Optional
    >>>
    >>> from sqlalchemy import Column
    >>> from sqlalchemy import create_engine
    >>> from sqlalchemy import ForeignKey
    >>> from sqlalchemy import Table
    >>> from sqlalchemy.orm import DeclarativeBase
    >>> from sqlalchemy.orm import Mapped
    >>> from sqlalchemy.orm import mapped_column
    >>> from sqlalchemy.orm import relationship
    >>> from sqlalchemy.orm import Session
    >>>
    >>>
    >>> class Base(DeclarativeBase):
    ...     pass
    >>> class User(Base):
    ...     __tablename__ = "user_account"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str]
    ...     fullname: Mapped[Optional[str]]
    ...     addresses: Mapped[List["Address"]] = relationship(back_populates="user")
    ...     orders: Mapped[List["Order"]] = relationship()
    ...
    ...     def __repr__(self) -> str:
    ...         return f"User(id={self.id!r}, name={self.name!r}, fullname={self.fullname!r})"
    >>> class Address(Base):
    ...     __tablename__ = "address"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     email_address: Mapped[str]
    ...     user: Mapped[User] = relationship(back_populates="addresses")
    ...
    ...     def __repr__(self) -> str:
    ...         return f"Address(id={self.id!r}, email_address={self.email_address!r})"
    >>> order_items_table = Table(
    ...     "order_items",
    ...     Base.metadata,
    ...     Column("order_id", ForeignKey("user_order.id"), primary_key=True),
    ...     Column("item_id", ForeignKey("item.id"), primary_key=True),
    ... )
    >>>
    >>> class Order(Base):
    ...     __tablename__ = "user_order"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     user_id: Mapped[int] = mapped_column(ForeignKey("user_account.id"))
    ...     items: Mapped[List["Item"]] = relationship(secondary=order_items_table)
    >>> class Item(Base):
    ...     __tablename__ = "item"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str]
    ...     description: Mapped[str]
    >>> engine = create_engine("sqlite+pysqlite:///:memory:", echo=True)
    >>> Base.metadata.create_all(engine)
    BEGIN ...
    >>> conn = engine.connect()
    >>> session = Session(conn)
    >>> session.add_all(
    ...     [
    ...         User(
    ...             name="海绵宝宝",
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
    ...             name="派大星",
    ...             fullname="Patrick Star",
    ...             addresses=[Address(email_address="pat999@aol.com")],
    ...         ),
    ...         User(
    ...             name="章鱼哥",
    ...             fullname="Squidward Tentacles",
    ...             addresses=[Address(email_address="stentcl@sqlalchemy.org")],
    ...         ),
    ...         User(name="Mr.Crabs", fullname="Eugene H. Krabs"),
    ...     ]
    ... )
    >>> session.commit()
    BEGIN ... COMMIT
    >>> conn.begin()
    BEGIN ...