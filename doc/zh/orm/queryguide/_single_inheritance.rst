单表继承 ORM 查询指南设置
================================

本页面说明了 :doc:`inheritance` 中单表继承部分的四个示例项目使用的映射和 Fixture 数据。

..  sourcecode:: python

    >>> from sqlalchemy import create_engine
    >>> from sqlalchemy import ForeignKey
    >>> from sqlalchemy.orm import DeclarativeBase
    >>> from sqlalchemy.orm import Mapped
    >>> from sqlalchemy.orm import mapped_column
    >>> from sqlalchemy.orm import relationship
    >>> from sqlalchemy.orm import Session
    >>>
    >>> class Base(DeclarativeBase):
    ...     pass
    >>> class Employee(Base):
    ...     __tablename__ = "employee"
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     name: Mapped[str]
    ...     type: Mapped[str]
    ...
    ...     def __repr__(self):
    ...         return f"{self.__class__.__name__}({self.name!r})"
    ...
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "employee",
    ...         "polymorphic_on": "type",
    ...     }
    >>> class Manager(Employee):
    ...     manager_name: Mapped[str] = mapped_column(nullable=True)
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "manager",
    ...     }
    >>> class Engineer(Employee):
    ...     engineer_info: Mapped[str] = mapped_column(nullable=True)
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "engineer",
    ...     }
    >>>
    >>> engine = create_engine("sqlite://", echo=True)
    >>>
    >>> Base.metadata.create_all(engine)
    BEGIN ...

    >>> conn = engine.connect()
    >>> session = Session(conn)
    >>> session.add_all(
    ...     [
    ...         Manager(
    ...             name="Mr. Krabs",
    ...             manager_name="Eugene H. Krabs",
    ...         ),
    ...         Engineer(name="SpongeBob", engineer_info="Krabby Patty Master"),
    ...         Engineer(
    ...             name="Squidward",
    ...             engineer_info="Senior Customer Engagement Engineer",
    ...         ),
    ...     ],
    ... )
    >>> session.commit()
    BEGIN ...
