关联代理
========

.. module:: sqlalchemy.ext.associationproxy

``associationproxy`` 用于创建关系中目标属性的读写视图。它实际上是隐藏了两个端点之间的“中间”属性的使用，并可用于从相关对象的集合或标量关系中挑选字段，或者减少使用关联对象模式时的冗长性。通过创造性的应用，关联代理允许构建几乎任何类型的几何体的复杂集合和字典视图，并使用标准的、透明配置的关系模式持久化到数据库中。

.. _associationproxy_scalar_collections:

简化标量集合
--------------

考虑两个类之间的多对多映射，即 ``User`` 和 ``Keyword``。每个 ``User`` 可以拥有任意数量的 ``Keyword`` 对象，反之亦然（多对多模式在   :ref:`relationships_many_to_many`  中有描述）。下面的示例以与前面相同的方式说明此模式，只是在 ` `User`` 类中添加了一个名为 ``User.keywords`` 的附加属性：

::

    from __future__ import annotations

    from typing import Final
    from typing import List

    from sqlalchemy import Column
    from sqlalchemy import ForeignKey
    from sqlalchemy import Integer
    from sqlalchemy import String
    from sqlalchemy import Table
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship
    from sqlalchemy.ext.associationproxy import association_proxy
    from sqlalchemy.ext.associationproxy import AssociationProxy


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(64))
        kw: Mapped[List[Keyword]] = relationship(secondary=lambda: user_keyword_table)

        def __init__(self, name: str):
            self.name = name

        # 从 'kw' 关系中代理 'keyword' 属性
        keywords: AssociationProxy[List[str]] = association_proxy("kw", "keyword")


    class Keyword(Base):
        __tablename__ = "keyword"
        id: Mapped[int] = mapped_column(primary_key=True)
        keyword: Mapped[str] = mapped_column(String(64))

        def __init__(self, keyword: str):
            self.keyword = keyword


    user_keyword_table: Final[Table] = Table(
        "user_keyword",
        Base.metadata,
        Column("user_id", Integer, ForeignKey("user.id"), primary_key=True),
        Column("keyword_id", Integer, ForeignKey("keyword.id"), primary_key=True),
    )

在上面的示例中，将   :func:`.association_proxy`  应用于 ` `User`` 类，以产生 ``kw`` 关系的“视图”，该视图显示与每个 ``Keyword`` 对象相关联的 ``.keyword`` 的字符串值。它还通过透明地创建新的 ``Keyword`` 对象来添加到集合中的字符串。如下所示：

    >>> user = User("jek")
    >>> user.keywords.append("cheese-inspector")
    >>> user.keywords.append("snack-ninja")
    >>> print(user.keywords)
    ['cheese-inspector', 'snack-ninja']

要理解其中的原理，首先回顾一下不使用 ``.keywords`` 关联代理时 ``User`` 和 ``Keyword`` 的行为。通常，读取和操作与 ``User`` 关联的“keyword”字符串集合需要从每个集合元素遍历到 ``.keyword`` 属性，这可能是笨拙的。下面的示例说明了不使用关联代理时应用的相同操作：

    >>> # 不使用关联代理的相同操作
    >>> user = User("jek")
    >>> user.kw.append(Keyword("cheese-inspector"))
    >>> user.kw.append(Keyword("snack-ninja"))
    >>> print([keyword.keyword for keyword in user.kw])
    ['cheese-inspector', 'snack-ninja']

由   :func:`.association_proxy`  函数生成的   :class:` .AssociationProxy`  对象是 `Python 描述符 <https://docs.python.org/howto/descriptor.html>`_ 的实例，并且不被   :class:`.Mapper`  以任何方式认为是“映射”的。因此，它始终在映射类的类定义中内联指示，而不管使用 Declarative 还是 Imperative 映射。

代理通过响应操作在基础映射属性或集合上运行，并且通过代理进行的更改立即在映射属性中显示，反之亦然。基础属性仍然完全可访问。

第一次访问时，关联代理对目标集合执行内省操作，以便其行为正确对应。例如，将本地代理属性作为集合（通常情况下）或标量引用，并且集合是否的行为类似于 set、list 或 dictionary 等都会考虑在内，以便代理可以像底层集合或属性一样工作。

.. _associationproxy_creator:

创建新值
^^^^^^^^^^^^^^^^^^^^^^

当关联代理拦截到列表 ``append()`` 事件（或集合 ``add()``, 字典 ``__setitem__()`` 或标量赋值事件）时，它使用其构造函数实例化一个新的 “中间” 对象，将给定值作为单个参数传递。在上面的示例中，类似于以下的操作:: 

    user.keywords.append("cheese-inspector")

被关联代理转换为以下操作:: 

    user.kw.append(Keyword("cheese-inspector"))

这个例子可以工作是因为我们为 ``Keyword`` 设计了一个接受单个位置参数 ``keyword`` 的构造函数。对于那些不可行的单参数构造函数的情况，可以使用  :paramref:`.association_proxy.creator`  参数自定义关联代理的创建性行为，它引用一个可调用对象（即Python函数），该可调用对象将在给定单个参数的情况下生成一个新对象实例。下面我们使用一个Lambda函数来说明这点，这是典型的做法:: 

    class User(Base):
        ...

        # 在 append() 事件中使用 Keyword(keyword=kw)
        keywords: AssociationProxy[List[str]] = association_proxy(
            "kw", "keyword", creator=lambda kw: Keyword(keyword=kw)
        )

在列表或集合类型的集合中，``creator`` 函数接受单个参数，对于标量属性也是如此。对于基于字典的集合，它接受两个参数，"key" 和 "value"。下面是在   :ref:`proxying_dictionaries`  中的一个例子。

简化关联对象
-----------------------

“关联对象”模式是一种扩展形式的多对多关系模式，如  :ref:`association_pattern` 所述。关联代理有助于在常规使用过程中避免“关联对象籍”干扰。

假设上面的“user_keyword”表有额外的列需要显式映射，但在大多数情况下我们不需要直接访问这些属性。下面，我们展示了一种新的映射方法，引入了 ``UserKeywordAssociation`` 类，该类映射到前面所述的 ``user_keyword`` 表。该类增加了一个附加列 ``special_key``，我们偶尔需要访问其值，但通常情况下不需要。我们在User类上创建了一个名为 ``keywords`` 的关联代理，它将从每个 ``UserKeywordAssociation`` 上存在的 ``keyword`` 属性来构建从 ``User`` 的 ``user_keyword_associations`` 集合到 ``.keyword`` 属性之间的桥梁::

    from __future__ import annotations

    from typing import List
    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy import String
    from sqlalchemy.ext.associationproxy import association_proxy
    from sqlalchemy.ext.associationproxy import AssociationProxy
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"

        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(64))

        user_keyword_associations: Mapped[List[UserKeywordAssociation]] = relationship(
            back_populates="user",
            cascade="all, delete-orphan",
        )

        # 关联代理收集“user_keyword_associations”到“keyword”属性
        keywords: AssociationProxy[List[Keyword]] = association_proxy(
            "user_keyword_associations",
            "keyword",
            creator=lambda keyword_obj: UserKeywordAssociation(keyword=keyword_obj),
        )

        def __init__(self, name: str):
            self.name = name


    class UserKeywordAssociation(Base):
        __tablename__ = "user_keyword"
        user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
        keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
        special_key: Mapped[Optional[str]] = mapped_column(String(50))

        user: Mapped[User] = relationship(back_populates="user_keyword_associations")

        keyword: Mapped[Keyword] = relationship()


    class Keyword(Base):
        __tablename__ = "keyword"
        id: Mapped[int] = mapped_column(primary_key=True)
        keyword: Mapped[str] = mapped_column("keyword", String(64))

        def __init__(self, keyword: str):
            self.keyword = keyword

        def __repr__(self) -> str:
            return f"Keyword({self.keyword!r})"

使用上面的配置，我们可以操作每个User对象的 ``.keywords`` 集合，每个集合都公开了从底层的 ``UserKeywordAssociation`` 元素中获得的 ``Keyword`` 对象集合:: 

    >>> user = User("log")
    >>> for kw in (Keyword("new_from_blammo"), Keyword("its_big")):
    ...     user.keywords.append(kw)
    >>> print(user.keywords)
    [Keyword('new_from_blammo'), Keyword('its_big')]

这个例子与之前在   :ref:`associationproxy_scalar_collections` ` .keywords.append()`` 操作都等同于:: 

    >>> user.user_keyword_associations.append(
    ...     UserKeywordAssociation(keyword=Keyword("its_heavy"))
    ... )

``UserKeywordAssociation`` 对象有两个属性都在关联代理的 ``append()`` 操作的范围内填充；``.keyword``，它引用 ``Keyword`` 对象，和 ``.user``， 它引用 ``User`` 对象。先填充``.keyword``属性，因为关联代理创建关系的方式是在套接字/外键上。在 ``.append()`` 操作的响应中，生成一个新的 ``UserKeywordAssociation`` 对象，将给定的 ``Keyword`` 实例分配给``.keyword`` 属性。然后， `UserKeywordAssociation` 对象附加到 ``User.user_keyword_associations`` 集合上， ``User.user_keyword_associations`` 关系都会初始化 ``UserKeywordAssociation.user`` 属性，这是为了引用接收附加操作的父 ``User``。上面的 ``special_key`` 参数保持其默认值 ``None``。

对于我们确实需要 ``special_key`` 具有某种值的情况，我们显式地创建 ``UserKeywordAssociation`` 对象。下面我们分配三个属性，其中构造期间的 ``.user`` 赋值的效果是通过关系将新的 ``UserKeywordAssociation`` 添加到``User.user_keyword_associations`` 集合中::

    >>> UserKeywordAssociation(
    ...     keyword=Keyword("its_wood"), user=user, special_key="my special key"
    ... )

关联代理为我们提供了 `Keyword` 对象的集合，表示所有这些操作::

    >>> print(user.keywords)
    [Keyword('new_from_blammo'), Keyword('its_big'), Keyword('its_heavy'), Keyword('its_wood')]

.. _proxying_dictionaries:

代理字典集合
----------------------------------------

关联代理也可以代理字典集合。SQLAlchemy 映射通常使用   :func:`.attribute_keyed_dict`  集合类型创建字典集合，以及   :ref:` dictionary_collections`  中描述的扩展技术。

当检测到使用基于字典的集合时，关联代理会调整其行为。当向字典添加新值时，代理将实例化中介对象，通过传递两个参数（键和值）而不是一个参数来创建中介对象。与往常一样，这个创建函数默认为中介类的构造函数，可以使用 ``creator`` 参数进行自定义。

下面，我们修改我们的 ``UserKeywordAssociation`` 示例，使得 ``User.user_keyword_associations`` 集合使用字典映射，其中 ``UserKeywordAssociation.special_key`` 参数将用作字典的键。我们还将 ``creator`` 参数应用于 ``User.keywords`` 代理，以便在向字典添加新元素时正确分配这些值::

    from __future__ import annotations
    from typing import Dict

    from sqlalchemy import ForeignKey
    from sqlalchemy import String
    from sqlalchemy.ext.associationproxy import association_proxy
    from sqlalchemy.ext.associationproxy import AssociationProxy
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship
    from sqlalchemy.orm.collections import attribute_keyed_dict


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(64))

        # user/user_keyword_associations relationship, mapping
        # user_keyword_associations with a dictionary against "special_key" as key.
        user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
            back_populates="user",
            collection_class=attribute_keyed_dict("special_key"),
            cascade="all, delete-orphan",
        )
        # proxy to 'user_keyword_associations', instantiating
        # UserKeywordAssociation assigning the new key to 'special_key',
        # values to 'keyword'.
        keywords: AssociationProxy[Dict[str, Keyword]] = association_proxy(
            "user_keyword_associations",
            "keyword",
            creator=lambda k, v: UserKeywordAssociation(special_key=k, keyword=v),
        )

        def __init__(self, name: str):
            self.name = name


    class UserKeywordAssociation(Base):
        __tablename__ = "user_keyword"
        user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
        keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
        special_key: Mapped[str]

        user: Mapped[User] = relationship(
            back_populates="user_keyword_associations",
        )
        keyword: Mapped[Keyword] = relationship()


    class Keyword(Base):
        __tablename__ = "keyword"
        id: Mapped[int] = mapped_column(primary_key=True)
        keyword: Mapped[str] = mapped_column(String(64))

        def __init__(self, keyword: str):
            self.keyword = keyword

        def __repr__(self) -> str:
            return f"Keyword({self.keyword!r})"

我们将 ``.keywords`` 集合说明为字典集合，将 ``UserKeywordAssociation.special_key`` 的值映射为 ``Keyword`` 对象::

    >>> user = User("log")

    >>> user.keywords["sk1"] = Keyword("kw1")
    >>> user.keywords["sk2"] = Keyword("kw2")

    >>> print(user.keywords)
    {'sk1': Keyword('kw1'), 'sk2': Keyword('kw2')}

.. _composite_association_proxy:

组合关联代理
-----------------------------

鉴于我们先前的示例是从关系到标量属性的代理、跨关联对象的代理和代理字典，我们可以将所有三种技术结合在一起，为 ``User`` 提供仅涉及 ``special_key`` 的字符串值映射到字符串 ``keyword`` 的 ``keywords``字典。``Keyword``类全部被隐藏起来了。这是通过在``User``上建立一个关联代理实现的，该代理引用了``UserKeywordAssociation``上存在的关联代理：

    from __future__ import annotations

    from sqlalchemy import ForeignKey
    from sqlalchemy import String
    from sqlalchemy.ext.associationproxy import association_proxy
    from sqlalchemy.ext.associationproxy import AssociationProxy
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship
    from sqlalchemy.orm.collections import attribute_keyed_dict


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(64))

        user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
            back_populates="user",
            collection_class=attribute_keyed_dict("special_key"),
            cascade="all, delete-orphan",
        )
        # 和基本字典示例中一样的'user_keyword_associations'->'keyword'代理。
        keywords: AssociationProxy[Dict[str, str]] = association_proxy(
            "user_keyword_associations",
            "keyword",
            creator=lambda k, v: UserKeywordAssociation(special_key=k, keyword=v),
        )

        def __init__(self, name: str):
            self.name = name


    class UserKeywordAssociation(Base):
        __tablename__ = "user_keyword"
        user_id: Mapped[int] = mapped_column(ForeignKey("user.id"), primary_key=True)
        keyword_id: Mapped[int] = mapped_column(ForeignKey("keyword.id"), primary_key=True)
        special_key: Mapped[str] = mapped_column(String(64))
        user: Mapped[User] = relationship(
            back_populates="user_keyword_associations",
        )

        # Keyword的关系现在称为'kw'
        kw: Mapped[Keyword] = relationship()

        # 'keyword'被改成与'Keyword'的'keyword'属性关联的代理。
        keyword: AssociationProxy[Dict[str, str]] = association_proxy("kw", "keyword")


    class Keyword(Base):
        __tablename__ = "keyword"
        id: Mapped[int] = mapped_column(primary_key=True)
        keyword: Mapped[str] = mapped_column(String(64))

        def __init__(self, keyword: str):
            self.keyword = keyword

``User.keywords``现在是一个字符串到字符串的字典，使用关联代理，``UserKeywordAssociation``和``Keyword``对象会自动为我们创建和删除。在下面的示例中，我们展示了使用赋值运算符的用法，同样被关联代理适当处理，一次向集合中添加字典值：

    >>> user = User("log")
    >>> user.keywords = {"sk1": "kw1", "sk2": "kw2"}
    >>> print(user.keywords)
    {'sk1': 'kw1', 'sk2': 'kw2'}

    >>> user.keywords["sk3"] = "kw3"
    >>> del user.keywords["sk2"]
    >>> print(user.keywords)
    {'sk1': 'kw1', 'sk3': 'kw3'}

    >>> # 展示未被代理的用法
    ... print(user.user_keyword_associations["sk3"].kw)
    <__main__.Keyword object at 0x12ceb90>

需要注意的是，上述示例中由于每次字典设置操作都会创建``Keyword``对象，这种方式对于标签场景通常要求``Keyword``对象的字符串名称保持唯一性。针对这种用例，建议使用`UniqueObject <https://www.sqlalchemy.org/trac/wiki/UsageRecipes/UniqueObject>`_ 或类似的构造策略，将“先查找，然后创建”策略应用到``Keyword``类的构造函数中，这样如果已经存在给定名称的``Keyword``对象，则返回该对象。

使用关联代理进行查询
---------------------------

  :class:`.AssociationProxy` ` EXISTS`` 关键字的过滤功能。


.. note::
    关联代理扩展的主要目的是允许改善已加载的映射对象实例的持久性和对象访问模式。类绑定的查询功能有限，无法替代在构造带有JOIN、预加载选项等SQL查询时引用底层属性的需要。

对于此部分，假设一个类既有关联到列的关联代理，又有关联到相关对象的关联代理，如下面的映射示例所示：

    from __future__ import annotations
    from sqlalchemy import Column, ForeignKey, Integer, String
    from sqlalchemy.ext.associationproxy import association_proxy, AssociationProxy
    from sqlalchemy.orm import DeclarativeBase, relationship
    from sqlalchemy.orm.collections import attribute_keyed_dict
    from sqlalchemy.orm.collections import Mapped


    class Base(DeclarativeBase):
        pass


    class User(Base):
        __tablename__ = "user"
        id: Mapped[int] = mapped_column(primary_key=True)
        name: Mapped[str] = mapped_column(String(64))

        user_keyword_associations: Mapped[UserKeywordAssociation] = relationship(
            cascade="all, delete-orphan",
        )

        # object-targeted association proxy
        keywords: AssociationProxy[Dict[str, str]] = association_proxy(
            "user_keyword_associations",
            "keyword",
        )

        # column-targeted association proxy关联代理（Association Proxy）
-----------------------------

关联代理是一种属性，通常在ORM模式下使用，它允许操作对象关系型数据库表的两个相关的列。 关联代理具有两个主要属性：它的内容和它的目标。

**内容**具有关系输出，因此它可以赋值和使用像Python对象一样。在下面的例子中，我们显示了如何用一个简单的关联代理属性：  :attr:`.User.keywords` ：

.. sourcecode:: python

    from sqlalchemy.orm import relationship, selectinload
    from sqlalchemy.ext.associationproxy import association_proxy

    class User(Base):
        __tablename__ = 'user'
        id = Column(Integer, primary_key=True)
        name = Column(String)
        keywords_association = relationship(
            'UserKeywordAssociation',
            back_populates='user', cascade='all, delete-orphan'
        )
        keywords = association_proxy(
            'keywords_association', 'keyword',
            creator=lambda kw: UserKeywordAssociation(keyword=kw)
        )

    class UserKeywordAssociation(Base):
        __tablename__ = 'user_keyword'
        user_id = Column(
            Integer,
            ForeignKey('user.id', ondelete='CASCADE'),
            primary_key=True
        )
        keyword_id = Column(
            Integer,
            ForeignKey('keyword.id', ondelete='CASCADE'),
            primary_key=True
        )
        keyword = relationship('Keyword')

    class Keyword(Base):
        __tablename__ = 'keyword'
        id = Column(Integer, primary_key=True)
        keyword = Column(String)

在这个例子中，在User模型上，我们定义了一个集合，即  property`User.keywords_associtation`， 以及一个关联代理称为  :attr:`.User.keywords` 。关联代理是使用   :func:` .association_proxy`  从来自 ``UserKeywordAssociation`` 的关联列表添加的。关联代理创建一个到 ``UserKeywordAssociation.keyword`` 的映射，这是一个字符串栏，它允许像列表属性一样对User的Keyword内容进行读写。 在下面这个示例中，我们可以通过查询公共关键字来搜索用户。

.. sourcecode:: pycon+sql

    >>> jek = User(name='jek')
    >>> jek.keywords.append('python')
    >>> session.add(jek)
    >>> session.commit()

    >>> print(session.scalars(select(User).where(User.keywords == 'python')))
    {printsql}SELECT user.id AS user_id, user.name AS user_name 
     FROM user 
     WHERE EXISTS (SELECT 1 
                   FROM user_keyword 
                   JOIN keyword ON keyword.id = user_keyword.keyword_id 
                   WHERE user_keyword.user_id = user.id AND keyword.keyword = :keyword_1)

``User.keywords`` 集群可以直接添加关键字：

.. sourcecode:: pycon+sql

    >>> jek.keywords.append('rest')
    >>> session.commit()

我们可以连接一个表达式，例如``LIKE``操作符：

.. sourcecode:: pycon+sql

    >>> jek.keywords.append('POST')
    >>> print(session.scalars(select(User).
           where(User.keywords.any("LIKE" + User.keywords.type.python_type('%post%')))))
    {printsql}SELECT user.id AS user_id, user.name AS user_name 
     FROM user 
     WHERE EXISTS (SELECT 1 
                   FROM user_keyword 
                   JOIN keyword ON keyword.id = user_keyword.keyword_id 
                   WHERE user_keyword.user_id = user.id 
                     AND keyword LIKE :keyword_1 ESCAPE '/')

在接下来的例子中，我们展示了如何在关联代理中使用 `ANY (value_clause) <http://docs.sqlalchemy.org/en/latest/core/sqlelement.html?highlight=any#sqlalchemy.sql.expression.ColumnOperators.any>`_。

假设我们已经定义了一个名称为 ``User.keywords`` 的关联代理属性，然后可以在查询中使用 ``user.keywords.any(value_clause=expression)`` 对每个子项目指定一个所需的筛选器。 在下面的示例中，我们需要使用 ``Keyword.keyword == "foo"`` 来过滤出所有使用``foo``关键字的用户。

.. sourcecode:: pycon+sql

    >>> # from sqlalchemy.sql.expression import subquery
    >>> inner_subq = (
    ...     select(UserKeywordAssociation.user_id).
    ...     join(Keyword).filter(Keyword.keyword == 'foo').subquery()
    ... )
    >>> print(session.query(User).filter(User.id.in_(
    ...     select(inner_subq)
    ...     .where(inner_subq.c.user_id == User.id)
    ... )))
    {printsql}SELECT user.id AS user_id, user.name AS user_name 
     FROM user 
     WHERE user.id IN (SELECT user_keyword.user_id AS user_keyword_user_id 
                       FROM user_keyword JOIN keyword ON keyword.id = user_keyword.keyword_id 
                       WHERE keyword.keyword = :keyword_1 INTERSECT SELECT user_id 
                                                                         FROM user_keyword)

在此示例中，我们首先查询子查询来确定所有使用 ``foo`` 关键字的 assoc/用户ID。 在使用查询 ``session.query（User）`` 进行帐户查询的上下文中，我们使用约束 `` User.id.in_`` 返回 ``User.id`` 集。 但是，我们将再次使用 ``User.id == inner_subq.c.user_id`` 对UserKeywordAssociation进行过滤。 首先，集合的操作允许我们使用SQL“INTERSECT”进行此过滤器。 然后，在查询的主体上下文中，我们可以使用任何其他过滤器来缩小。

这些查询将生成一些内部的含子查询，所以在进行关联代理查询时要遵循常见的SQL性能提示，例如避免在子查询中使用聚合。

.. note:: 

    与标量关联代理的主要区别是，使用标量关联代理，只能访问一对一关系，而使用键控关联代理，可以访问关系列表。从sqlalchemy.ext.associationproxy中导入AssociationProxy
从sqlalchemy.orm中导入DeclarativeBase、Mapped、mapped_column、relationship

class Base(DeclarativeBase):
    pass

class Recipe(Base):
    __tablename__ = "recipe"
    id: Mapped[int] = mapped_column(primary_key=True)
    name: Mapped[str] = mapped_column(String(64))

    steps: Mapped[List[Step]] = relationship(back_populates="recipe")
    step_descriptions: AssociationProxy[List[str]] = association_proxy(
        "steps", "description"
    )


class Step(Base):
    __tablename__ = "step"
    id: Mapped[int] = mapped_column(primary_key=True)
    description: Mapped[str]
    recipe_id: Mapped[int] = mapped_column(ForeignKey("recipe.id"))
    recipe: Mapped[Recipe] = relationship(back_populates="steps")

    recipe_name: AssociationProxy[str] = association_proxy("recipe", "name")

    def __init__(self, description: str) -> None:
        self.description = description

my_snack = Recipe(
    name="下午点心",
    step_descriptions=[
        "切面包",
        "涂上花生酱",
        "吃三明治",
    ],
)

使用以下方式可以打印“my_snack”的步骤概述::

    >>> for i, step in enumerate(my_snack.steps, 1):
    ...     print(f"{step.recipe_name!r}的第{i}步：{step.description}")
    '下午点心'的第1步：切面包
    '下午点心'的第2步：涂上花生酱
    '下午点心'的第3步：吃三明治

API文档
-----------------

.. autofunction:: association_proxy

.. autoclass:: AssociationProxy
   :members:
   :undoc-members:
   :inherited-members:

.. autoclass:: AssociationProxyInstance
   :members:
   :undoc-members:
   :inherited-members:

.. autoclass:: ObjectAssociationProxyInstance
   :members:
   :inherited-members:

.. autoclass:: ColumnAssociationProxyInstance
   :members:
   :inherited-members:

.. autoclass:: AssociationProxyExtensionType
   :members: