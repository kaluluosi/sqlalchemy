ORM查询指南设置：单继承
=========================

该页面展示了在  :doc:`继承`  文档的   :ref:` single_inheritance`  示例中所使用的映射和装置数据。

.. sourcecode:: python


    >>> from sqlalchemy import create_engine
    >>> from sqlalchemy import ForeignKey
    >>> from sqlalchemy.orm import DeclarativeBase
    >>> from sqlalchemy.orm import Mapped
    >>> from sqlalchemy.orm import mapped_column
    >>> from sqlalchemy.orm import relationship
    >>> from sqlalchemy.orm import Session
    >>>
    >>> # 创建Declarative基类
    >>> class Base(DeclarativeBase):
    ...     pass

    >>> # 创建员工类
    >>> class Employee(Base):
    ...     __tablename__ = "employee"
    ...     
    ...     # 主键id
    ...     id: Mapped[int] = mapped_column(primary_key=True)
    ...     
    ...     # 姓名
    ...     name: Mapped[str]
    ...     
    ...     # 职工类型
    ...     type: Mapped[str]
    ...
    ...     def __repr__(self):
    ...         return f"{self.__class__.__name__}({self.name!r})"
    ...
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "employee",
    ...         "polymorphic_on": "type",
    ...     }

    >>> # 创建经理类，继承自员工类
    >>> class Manager(Employee):
    ...     
    ...     # 经理姓名
    ...     manager_name: Mapped[str] = mapped_column(nullable=True)
    ...
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "manager",
    ...     }

    >>> # 创建工程师类，继承自员工类
    >>> class Engineer(Employee):
    ...     
    ...     # 工程师信息
    ...     engineer_info: Mapped[str] = mapped_column(nullable=True)
    ...
    ...     __mapper_args__ = {
    ...         "polymorphic_identity": "engineer",
    ...     }

    >>> # 数据库引擎
    >>> engine = create_engine("sqlite://", echo=True)
    >>>
    >>> # 创建表
    >>> Base.metadata.create_all(engine)
    BEGIN ...

    >>> # 数据库连接和会话
    >>> conn = engine.connect()
    >>> session = Session(conn)

    >>> # 添加数据
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

    >>> # 提交修改
    >>> session.commit()
    BEGIN ...