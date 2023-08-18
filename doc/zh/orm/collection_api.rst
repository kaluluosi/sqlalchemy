.. highlight:: python

.. _custom_collections_toplevel:

.. currentmodule:: sqlalchemy.orm

========================================
集合订制和API详细信息
========================================

:func:`_orm.relationship`函数定义了两个类之间的联系。当联系定义为一对多或多对多关系时,加载和操作对象时,它会被表示为Python集合。本部分介绍集合配置和技巧的额外信息。

.. _custom_collections:

定制集合访问
-----------------------------

在一个类映射成一对多或多对多关系之后，通过父实例上的属性访问一个值的集合。这两个常见的集合类型是 ``list`` 和 ``set``，当使用 :class:`_orm.Mapped`  的 :ref:`声明性映射 <orm_declarative_styles_toplevel>` 时，使用集合类型在 :class:`_orm.Mapped` 容器内部进行设置，如下所示 ``Parent.children`` 集合，其中使用的是 ``list``：

    from sqlalchemy import ForeignKey

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Parent(Base):
        __tablename__ = "parent"

        parent_id: Mapped[int] = mapped_column(primary_key=True)

        # 使用列表
        children: Mapped[List["Child"]] = relationship()


    class Child(Base):
        __tablename__ = "child"

        child_id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))

或针对一个 ``set``，在同样的“Parent.children”集合中演示：

    from typing import Set
    from sqlalchemy import ForeignKey

    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Parent(Base):
        __tablename__ = "parent"

        parent_id: Mapped[int] = mapped_column(primary_key=True)

        # 使用 set
        children: Mapped[Set["Child"]] = relationship()


    class Child(Base):
        __tablename__ = "child"

        child_id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))

.. note::  如果使用Python 3.7或3.8，集合的注释需要使用 ``typing.List`` 或 ``typing.Set``，例如 ``Mapped[List["Child"]]`` 或 ``Mapped[Set["Child"]]``；因为这些Python版本中的 ``list`` 和 ``set`` Python内置不支持通用注释。

当使用没有 :class:`_orm.Mapped` 注释的映射时，例如在 :ref:`命令式映射 <orm_imperative_mapping>` 或未打了类型标签的 Python 代码中，以及一些特殊情况下，可以使用 :paramref:`_orm.relationship.collection_class` 参数直接指定 :func:`_orm.relationship` 的集合类：

    # 非注释映射


    class Parent(Base):
        __tablename__ = "parent"

        parent_id = mapped_column(Integer, primary_key=True)

        children = relationship("Child", collection_class=set)


    class Child(Base):
        __tablename__ = "child"

        child_id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(ForeignKey("parent.id"))

在缺乏 :paramref:`_orm.relationship.collection_class` 或 :class:`_orm.Mapped` 的情况下，默认集合类型是 ``list``。
除了构建在内的 ``list`` 和 ``set`` 之外，还支持两种字典类型，下面的章节 :ref:`orm_dictionary_collection` 中对此进行了描述。还可以将任何任意的可变序列类型设置为目标集合，但需要进行一些附加配置步骤；在章节 :ref:`orm_custom_collection` 中进行描述。


.. _orm_dictionary_collection:

字典集合订制
~~~~~~~~~~~~~~~~~~~~~~

使用字典作为集合需要一些额外细节。因为对象总是以列表的形式从数据库中加载的，所以必须有一种键生成策略才能正确填充字典。 :func:`.attribute_keyed_dict` 函数是实现简单字典集合的最常见方式。它生成一个字典类，该字典类将一个特定属性的映射类作为键。例如，下面映射了一个包含按“Note.keyword”属性标记的“Note”项的字典的“Item”类。当使用 :func:`.attribute_keyed_dict` 时， :class:`_orm.Mapped` 可以使用 :class:`_orm.KeyFuncDict` 或普通 ``dict``，如下所示。但是，在此情况下需要 :paramref:`_orm.relationship.collection_class` 参数以便可以适当地构造 :func:`.attribute_keyed_dict` 。：

    from typing import Dict
    from typing import Optional

    from sqlalchemy import ForeignKey
    from sqlalchemy.orm import attribute_keyed_dict
    from sqlalchemy.orm import DeclarativeBase
    from sqlalchemy.orm import Mapped
    from sqlalchemy.orm import mapped_column
    from sqlalchemy.orm import relationship


    class Base(DeclarativeBase):
        pass


    class Item(Base):
        __tablename__ = "item"

        id: Mapped[int] = mapped_column(primary_key=True)

        notes: Mapped[Dict[str, "Note"]] = relationship(
            collection_class=attribute_keyed_dict("keyword"),
            cascade="all, delete-orphan",
        )


    class Note(Base):
        __tablename__ = "note"

        id: Mapped[int] = mapped_column(primary_key=True)
        item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
        keyword: Mapped[str]
        text: Mapped[Optional[str]]

        def __init__(self, keyword: str, text: str):
            self.keyword = keyword
            self.text = text

然后， ``Item.notes`` 是一个字典：

    >>> item = Item()
    >>> item.notes["a"] = Note("a", "atext")
    >>> item.notes.items()
    {'a': <__main__.Note object at 0x2eaaf0>}

:func:`.attribute_keyed_dict`  将确保每个 ``Note`` 的 ``.keyword`` 属性符合字典中的键。例如，在“Item.notes”中指定键时，我们提供的字典键必须与实际 ``Note`` 对象的键匹配：

```
item = Item()
item.notes = {
    "a": Note("a", "atext"),
    "b": Note("b", "btext"),
}
```

``.attribute_keyed_dict`` 使用的作为键的属性完全不需要被映射！  使用常规Python ``@property`` 允许用于将对象的任何细节或组合用作键，正如在此之前所述的属性 ``Note.keyword`` 和 ``Note.text`` 字段的前十个字母的元组中所建立的那样：

    class Item(Base):
        __tablename__ = "item"

        id: Mapped[int] = mapped_column(primary_key=True)

        notes: Mapped[Dict[str, "Note"]] = relationship(
            collection_class=attribute_keyed_dict("note_key"),
            back_populates="item",
            cascade="all, delete-orphan",
        )


    class Note(Base):
        __tablename__ = "note"

        id: Mapped[int] = mapped_column(primary_key=True)
        item_id: Mapped[int] = mapped_column(ForeignKey("item.id"))
        keyword: Mapped[str]
        text: Mapped[str]

        item: Mapped["Item"] = relationship()

        @property
        def note_key(self):
            return (self.keyword, self.text[0:10])

        def __init__(self, keyword: str, text: str):
            self.keyword = keyword
            self.text = text

上面上述的代码添加了 ``Note.item`` 关系，具有双向的 :paramref:`_orm.relationship.back_populates` 配置。将一些反向关系添加到此，``Note`` 将添加到 ``Item.notes`` 字典中，并自动为我们生成键：

    >>> item = Item()
    >>> n1 = Note("a", "atext")
    >>> n1.item = item
    >>> item.notes
    {('a', 'atext'): <__main__.Note object at 0x2eaaf0>}

其他内置字典类型包括 :func:`.column_keyed_dict`，它几乎与 :func:`.attribute_keyed_dict` 相同，除了直接给出了 :class:`_schema.Column` 对象；以及 :func:`.mapped_collection`，它传递任何可调用函数。请注意，通常建议结合使用 :func:`.attribute_keyed_dict` 和 ``@property``，如前面提到的。

字典映射通常与“Association Proxy”扩展结合使用以产生流线型字典视图。请参阅 :ref:`proxying_dictionaries` 和 :ref:`composite_association_proxy` 以获取样例。

.. _key_collections_mutations:

解决键变异和针对字典集合的反向填充
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

当使用 :func:`.attribute_keyed_dict` 时，字典的“键”取自目标对象上的一个属性。**这些键的更改不会被跟踪**。这意味着必须在首次使用它们时分配键，如果键更改，集合将不会发生变化。一个典型的例子是依靠反参考关系填充映射属性集合。以下是一个示例：

    class A(Base):
        __tablename__ = "a"

        id: Mapped[int] = mapped_column(primary_key=True)

        bs: Mapped[Dict[str, "B"]] = relationship(
            collection_class=attribute_keyed_dict("data"),
            back_populates="a",
        )


    class B(Base):
        __tablename__ = "b"

        id: Mapped[int] = mapped_column(primary_key=True)
        a_id: Mapped[int] = mapped_column(ForeignKey("a.id"))
        data: Mapped[str]

        a: Mapped["A"] = relationship(back_populates="bs")

以上代码若使用 ``B()`` 指定特定的 ``A()``，则反填将向 ``A.bs`` 集合中添加 ``B()``，但如果 ``B.data`` 的值尚未设置，则键将为``None``：

    >>> a1 = A()
    >>> b1 = B(a=a1) # 在此之后才为 'data' 赋值
    >>> a1.bs
    {None: <test3.B object at 0x7f7b1023ef70>}

设置 ``b1.data`` 之后并不会更新集合：

    >>> b1.data = "the key"
    >>> a1.bs
    {None: <test3.B object at 0x7f7b1023ef70>}

如果试图在构造函数中设置 ``B()``，也会发现这一点。使用 ``B.data`` 的值在构造函数内部作为键会改变结果：

    >>> B(a=a1, data="the key")
    <test3.B object at 0x7f7b10114280>
    >>> a1.bs
    {None: <test3.B object at 0x7f7b10114280>}

vs：

    >>> B(data="the key", a=a1)
    <test3.B object at 0x7f7b10114340>
    >>> a1.bs
    {'the key': <test3.B object at 0x7f7b10114340>}

如果使用反参考关系，确保通过 ``__init__`` 方法按正确顺序填充属性。

类似下面的事件处理程序也可用于跟踪集合中的变化：

    from sqlalchemy import event
    from sqlalchemy.orm import attributes


    @event.listens_for(B.data, "set")
    def set_item(obj, value, previous, initiator):
        if obj.a is not None:
            previous = None if previous == attributes.NO_VALUE else previous
            obj.a.bs[value] = obj
            obj.a.bs.pop(previous)

.. _orm_custom_collection:

定制集合实现
---------------------------------

您也可以使用自己的类型作为集合。对于简单情况，继承自 ``list`` 或 ``set``，并添加自定义行为就足够了。在其他情况下，需要特殊的装饰器以告诉SQLAlchemy更多关于集合运作的细节。

.. topic:: 我需要一个定制的集合实现吗？

   在大多数情况下，不需要！使用“自验证”或者简单划转支持Struct的值的情况下，使用库内置的 ``list`` 或 ``set`` 并没有毛病。在这些基础上提供的装饰器能让您使用更多的订制选项。 有时，需要在外部建模的情况下，集合需要在访问或变异操作时具有特殊的行为，这时就需要进行个性化集合的处理。当然，可以将其与上述两种方法结合使用。

QLAlchemy中的集合被透明地*仪器化*。仪器化意味着对集合的正常操作进行跟踪，并在刷新时将更改写入数据库。此外，集合操作可以触发*事件*，指示必须执行某些次要操作。二次操作的示例包括将子项保存在 :class:`~sqlalchemy.orm.session.Session` 的父级中（即 ``save-update`` 级联），以及同步双向关系的状态（即 :func:`.backref`）。

“集合”模块了解列表，集合和字典的基本接口，并自动应用支持这些内置操作的修饰符。实现基本集合接口的对象派生类型通过鸭子类型进行检测和识别：

.. sourcecode:: python+sql

    class ListLike:
        def __init__(self):
            self.data = []

        def append(self, item):
            self.data.append(item)

        def remove(self, item):
            self.data.remove(item)

        def extend(self, items):
            self.data.extend(items)

        def __iter__(self):
            return iter(self.data)

        def foo(self):
            return "foo"

``append``，``remove``和``extend``是已知的 ``list`` 成员，并将自动附加修饰符。``__iter__``不是修改方法，不会被仪器化，并且``foo``也不会被修饰。

鸭子类型（即猜测）当然并不可靠，因此可以通过提供``__emulates__``类属性来显式说明您正在实施的接口：

    class SetLike:
        __emulates__ = set

        def __init__(self):
            self.data = set()

        def append(self, item):
            self.data.add(item)

        def remove(self, item):
            self.data.remove(item)

        def __iter__(self):
            return iter(self.data)

此类看起来与 Python 的 ``list`` （即 "list-like"）类似，因为它有公共的 ``append`` 方法，但``__emulates__``特性强制它被视为 ``set``。已知 ``set`` 的部分接口包括 ``remove``，因此将自动为该方法添加修饰符。

但是这个类还不能正常工作：需要一些黏合剂来将其适配到SQLAlchemy中使用。ORM需要知道哪些方法用于附加，删除和迭代集合的成员。使用类似一个 ``list`` 或 ``set`` 的类型时，适当的方法已经很清楚了，并且会被自动使用。但是，上面的类只大致类似于一个 ``set``，并没有为SQLAlchemy提供可以默认使用的明确的适配方法。需要使用 :paramref:`_orm.relationship.collection_class` 参数在这种情况下使用其他方法。

使用修饰器 ``@collection.appender`` 可以像这样将问题解决，因为它 in ``SetLike`` class 并没有提供``add`` 的方法:

    from sqlalchemy.orm.collections import collection


    class SetLike:
        __emulates__ = set

        def __init__(self):
            self.data = set()

        @collection.appender
        def append(self, item):
            self.data.add(item)

        def remove(self, item):
            self.data.remove(item)

        def __iter__(self):
            return iter(self.data)

就像默认方法一样， ``append`` 和 ``remove`` 方法需传入一个映射实体作为单个参数，迭代器方法则直接调用而无需参数。

.. _dictionary_collections:

自定义字典型集合
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

:class:`.KeyFuncDict` 类可以作为自定义类型的基类或混入，以快速将“字典”集合支持添加到其他类中。它使用一个关键函数将委托到 ``__setitem__``  和 ``__delitem__``  进行设置：

    from sqlalchemy.orm.collections import KeyFuncDict


    class MyNodeMap(KeyFuncDict):
        """Holds 'Node' objects, keyed by the 'name' attribute."""

        def __init__(self, *args, **kw):
            super().__init__(keyfunc=lambda node: node.name)
            dict.__init__(self, *args, **kw)

在子类化 :class:`.KeyFuncDict` 时，必须使用 :meth:`.collection.internally_instrumented` 修饰符装饰自定义版本的 ``__setitem__()``
或 ``__delitem__()``，**如果**它们在该类的这些相同方法中调用了 :class:`.KeyFuncDict`的这些方法。因为 :class:`.KeyFuncDict` 方法已经被仪器化 - 从仪器化调用中调用它们可能会导致事件被重复触发或不恰当地触发，在罕见的情况下可能导致的内部状态破坏：

    from sqlalchemy.orm.collections import KeyFuncDict, collection


    class MyKeyFuncDict(KeyFuncDict):
        """使用 @internally_instrumented 当您的方法 调用到已仪器化的方法"""

        @collection.internally_instrumented
        def __setitem__(self, key, value, _sa_initiator=None):
            # do something with key, value
            super(MyKeyFuncDict, self).__setitem__(key, value, _sa_initiator)

        @collection.internally_instrumented
        def __delitem__(self, key, _sa_initiator=None):
            # do something with key
            super(MyKeyFuncDict, self).__delitem__(key, _sa_initiator)

ORM了解所有列表，集合和字典的基本接口，并将自动应用装置，使这些内置类型和它们的子类在强类型框架内使用更加安全。实现基本集合接口的对象派生类型通过鸭子类型进行识别,详见上文。用于附加和删除的默认方法必须由 ``appender`` 和 ``remover`` 方法修饰，但是在基本字典介面中没有兼容的方法可供SQLAlchemy默认使用。除非另有装饰，否则迭代将通过 ``values()``执行。如果您选择子类化 ``dict`` 或通过另一种方式提供向字典一样的集合行为，ORM将了解“字典”接口，并在使用 ``relationship（…）`` 中的 ``collection_class`` 参数进行适当仪器化。必须装修 `appender`和`remover`方法。

集合API
-----------------------------

.. currentmodule:: sqlalchemy.orm

.. autofunction:: attribute_keyed_dict

.. autofunction:: column_keyed_dict

.. autofunction:: keyfunc_mapping

.. autodata:: attribute_mapped_collection

.. autodata:: column_mapped_collection

.. autodata:: mapped_collection

.. autoclass:: sqlalchemy.orm.KeyFuncDict
   :members:

.. autodata:: sqlalchemy.orm.MappedCollection


集合内部
-----------------------------

.. currentmodule:: sqlalchemy.orm.collections

.. autofunction:: bulk_replace

.. autoclass:: collection
    :members:

.. autodata:: collection_adapter

.. autoclass:: CollectionAdapter

.. autoclass:: InstrumentedDict

.. autoclass:: InstrumentedList

.. autoclass:: InstrumentedSet

.. autofunction:: prepare_instrumentation