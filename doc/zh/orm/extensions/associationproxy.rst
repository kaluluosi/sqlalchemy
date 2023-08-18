关联代理
=================

.. module:: sqlalchemy.ext.associationproxy

``associationproxy`` 用于创建跨关系目标属性的读/写视图。 在两个端点之间隐藏了“中间”属性的使用，并且可以用于从相关对象的集合或标量关系中挑选字段，或者减少使用关联对象模式的冗长。 创新应用关联代理允许构建几乎任何几何形状的复杂集合和字典视图，并使用标准、透明地配置的关系模式存储在数据库中。

简化标量集合
------------------------------

考虑两个类 ``User`` 和 ``Keyword`` 之间的多对多映射。 每个 ``User`` 可以有任意数量的 ``Keyword`` 对象，反之亦然（多对多模式在 :ref:`relationships_many_to_many` 中描述）。下面的示例以相同的方式说明了这种模式，唯一的区别是在 ``User`` 类中添加了一个名为 ``User.keywords`` 的额外属性::

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

        # 从 'kw' 关系代理关键字属性
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

在上面的示例中，将 :func:`.association_proxy` 应用于 ``User`` 类，以产生 ``kw`` 关系的“视图”，该视图公开与每个 ``Keyword`` 对象相关联的字符串值。它还创建了新的 ``Keyword`` 对象，当字符串添加到集合中时，会自动得到透明地添加到集合中：

    >>> user = User("jek")
    >>> user.keywords.append("cheese-inspector")
    >>> user.keywords.append("snack-ninja")
    >>> print(user.keywords)
    ['cheese-inspector', 'snack-ninja']

要了解此操作的原理，请首先查看不使用 ``.keywords`` 关联代理的情况下 ``User`` 和 ``Keyword`` 的行为。通常，需要从每个集合元素遍历到 ``.keyword`` 属性来读取和操作与 ``User`` 相关的“关键字”字符串集合，这可能很麻烦。下面的示例说明了在不使用关联代理的情况下应用相同的操作：

    >>> # 不使用关联代理的相同操作
    >>> user = User("jek")
    >>> user.kw.append(Keyword("cheese-inspector"))
    >>> user.kw.append(Keyword("snack-ninja"))
    >>> print([keyword.keyword for keyword in user.kw])
    ['cheese-inspector', 'snack-ninja']

由 :func:`.association_proxy` 生成的 :class:`.AssociationProxy` 对象是 Python 描述符的一个实例，不被 :class:`.Mapper` 认为是映射对象的一部分。因此，即使使用 Declarative 或 Imperative 映射，它仍然在类定义内内联指示。

代理通过在响应操作上对底层映射属性或集合进行操作，并且通过代理进行的更改立即在映射属性中显现，反之亦然。底层属性仍然完全可访问。

第一次访问时，关联代理对目标集合执行内省操作，以使其行为正确。因此，它会考虑细节，例如本地代理属性是否是集合（通常是集合）还是标量引用，集合是否像集合、列表或字典一样工作，这样代理应该与基础集合或属性一样工作。

创建新值
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当捕获到 list ``append()`` 事件（或 set ``add()``, dictionary ``__setitem__()`` 或标量赋值事件）时，代理关系会使用其构造函数对中介对象进行实例化，将给定值作为单个参数传递。 在上面的示例中，如执行以下操作：

    user.keywords.append("cheese-inspector")

代理关系根据上述操作将其转换成：

    user.kw.append(Keyword("cheese-inspector"))

此示例之所以可行，是因为我们已经为 ``Keyword`` 的构造函数设计了一个单个位置参数 ``keyword``。对于那些不适合单参数构造函数的情况，可以使用 :paramref：`。association_proxy.creator`（参考）参数自定义关联代理的创建行为，该参数引用一个可调用对象（即 Python 函数），该可调用对象将给定的单数参数转换为一个新的对象实例。下面使用 Lambda 来说明此问题：

    class User(Base):
        ...
        # 使用 Keyword(keyword = kw)增加 () 事件
        keywords: AssociationProxy[List[str]] = association_proxy(
            "kw", "keyword", creator=lambda kw: Keyword(keyword=kw)
        )

在列表或 set- 基础集合的情况下， ``creator`` 函数接受一个参数。对于基于字典的集合，它接受两个参数（“键”和“值”）。下面是 :ref:`proxying_dictionaries` 中的示例。

简化关联对象
-------------------------------

替代“关联对象”模式是多对多关系的扩展形式，该模式在 :ref:`association_pattern` 中描述，关联代理有助于在常规使用中避免“关联对象”。

假设上面的 ``user_keyword`` 表格还有其他列需要显式映射，但在大多数情况下我们不需要直接访问这些属性。下面我们说明一个新的映射，引入了 ``UserKeywordAssociation`` 类，该类映射到上面所示的 ``user_keyword`` 表。此类添加了一个附加列``special_key``，我们偶尔需要访问该值，但通常情况下不需要访问该值。我们在 ``User`` 类上创建了一个名为``keywords``的关联代理，该代理将从``User.user_keyword_associations``集合与存储在每个``UserKeywordAssociation``对象中存在的``.keyword``属性桥接差异。

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

        # 'user_keyword_associations' 集合的代理
        # 到'keyword'属性
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

通过上述配置，我们可以在每个 ``User`` 对象上操作 ``.keywords`` 集合，其中每个集合都公开了一个 ``Keyword`` 集合，通过底层的 ``UserKeywordAssociation`` 元素获得。在下面的示例中，我们说明了使用赋值运算符，这也能被关联代理透明地处理，来一次性为集合应用字典值：

    >>> user = User("log")
    >>> for kw in (Keyword("new_from_blammo"), Keyword("its_big")):
    ...     user.keywords.append(kw)
    >>> print(user.keywords)
    [Keyword('new_from_blammo'), Keyword('its_big')]

与 :ref:`associationproxy_scalar_collections` 中示例相反，其中关联代理暴露了字符串集合，而不是组合对象集合。在此示例中，每个``.keywords.append()`` 操作等同于：

    >>> user.user_keyword_associations.append(
    ...     UserKeywordAssociation(keyword=Keyword("its_heavy"))
    ... )

``UserKeywordAssociation`` 对象具有两个属性，在关联代理目标的归属作用域内均被填充；``.keyword``属性引用``Keyword`` 对象，``.user``属性引用 ``User`` 对象，对于给定的 ``UserKeywordAssociation`` 实例，这些属性会在给定关联代理后生成。关键词上面的 ``special_key`` 参数不变，仍然是缺省值 ``None``。

对于那些我们确实希望 ``special_key`` 具有值的情况，我们可以显式地创建 ``UserKeywordAssociation`` 对象。下面我们分配所有三个属性值，其中在构建期间分配``.user``变量，具有将新的 ``UserKeywordAssociation`` 附加到 ``User.user_keyword_associations`` 集合（通过关系）的效果：

    >>> UserKeywordAssociation(
    ...     keyword=Keyword("its_wood"), user=user, special_key="my special key"
    ... )

关联代理将返回用所有这些操作所代表的 ``Keyword`` 集合：

    >>> print(user.keywords)
    [Keyword('new_from_blammo'), Keyword('its_big'), Keyword('its_heavy'), Keyword('its_wood')]

代理字典的传递
----------------------------------------

关联代理也可以代理到基于字典的集合。SQLAlchemy 映射通常使用 :func:`.attribute_keyed_dict` 集合类型来创建字典集合，以及在 :ref:`dictionary_collections` 中描述的扩展技术。

当关联代理检测到使用基于字典的集合时，它会调整其行为。当向字典添加新值时，代理关系通过传递两个参数到创建函数而不是一个参数来实例化中介对象，这是创建函数的默认行为，可以使用 ``creator`` 参数进行自定义。

下面，我们修改上面的 ``UserKeywordAssociation`` 示例，以便现在将 ``User.user_keyword_associations`` 集合映射到字典中，其中 ``UserKeywordAssociation.special_key`` 参数将用作键。我们还将在 ``User.keywords`` 代理上应用 ``creator`` 参数，以便在向字典添加新元素时正确分配这些值：

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

        # 将 user_keyword_associations 映射为字典 对'key'使用 UserKeywordAssociation.special_key 的key
        user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
            back_populates="user",
            collection_class=attribute_keyed_dict("special_key"),
            cascade="all, delete-orphan",
        )
        # 将'user_keyword_associations' -> 'keyword' 代理像在基本字典示例中一样
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

我们将 ``.keywords`` 集合说明为字典，将 ``UserKeywordAssociation.special_key`` 值映射到 ``Keyword`` 对象：

    >>> user = User("log")

    >>> user.keywords["sk1"] = Keyword("kw1")
    >>> user.keywords["sk2"] = Keyword("kw2")

    >>> print(user.keywords)
    {'sk1': Keyword('kw1'), 'sk2': Keyword('kw2')}

组合关联代理
-----------------------------

鉴于我们之前的关于从关系到标量属性代理以及跨关联对象代理从字典到集合的示例，我们可以将这三个技术结合在一起，给``User``添加一组仅与 ``special_key`` 映射到字符串。关联代理实际上是定位于 ``User`` 上，该代理引用了出现在 ``UserKeywordAssociation`` 上的另一个关联代理。

    from __future__ import annotations

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

        user_keyword_associations: Mapped[Dict[str, UserKeywordAssociation]] = relationship(
            back_populates="user",
            collection_class=attribute_keyed_dict("special_key"),
            cascade="all, delete-orphan",
        )

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

        kw: Mapped[Keyword] = relationship()

        keyword: AssociationProxy[Dict[str, str]] = association_proxy("kw", "keyword")


    class Keyword(Base):
        __tablename__ = "keyword"
        id: Mapped[int] = mapped_column(primary_key=True)
        keyword: Mapped[str] = mapped_column(String(64))

        def __init__(self, keyword: str):
            self.keyword = keyword

通过上面的操作，``User.keywords`` 现在是一个字符串到字符串的字典，在访问标量对象的属性时，关联代理是两个代理关系链接在一起的，所以当使用该代理来生成 SQL 语句时，我们会得到两个级别的 EXISTS 子查询。

使用 :paramref:`.AssociationProxy.cascade_scalar_deletes` 时，可以将标量关联修改为处理级联删除，以使设置 ``A.b`` 为 ``None`` 时，也会删除 ``A.ab``：

    a.b = None

如果没有设置 :paramref:`.AssociationProxy.cascade_scalar_deletes` 参数，那么上述关联对象 ``ia.ab`` 将仍保持不变。注意，这不是针对基于集合的关联代理的行为；在这种情况下，始终在删除代理集合成员时删除中介关联对象。行是否删除取决于关系级联设置。class Recipe(Base):
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
        name="下午小吃",
        step_descriptions=[
            "切面包",
            "涂花生酱",
            "吃三明治",
        ],
    )

要打印“my_snack”的步骤摘要，可以使用：

    >>> for i, step in enumerate(my_snack.steps, 1):
    ...     print(f"第{i}步 {step.recipe_name!r}: {step.description}")
    第1步 '下午小吃': 切面包
    第2步 '下午小吃': 涂花生酱
    第3步 '下午小吃': 吃三明治

API 文档
--------

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