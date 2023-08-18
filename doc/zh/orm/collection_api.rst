.. highlight:: python

.. _custom_collections_toplevel:

.. currentmodule:: sqlalchemy.orm

========================================
集合自定义和API详细信息
========================================

 :func:`_orm.relationship` 函数定义了两个类之间的链接。当联系定义一个一对多或多对多的关系时，当对象被加载和操作时，它会被表示为Python集合。本节提供关于集合配置和技术的附加信息。


.. _custom_collections:

定制集合访问
-----------------------------

将一个一对多或多对多的关系映射到一个父实例的属性上得到一个值的集合，可以通过这个属性进行访问。这两个基本的集合类型是列表和集合，在使用 ``_orm.Mapped`` 的   :ref:`Declarative <orm_declarative_styles_toplevel>`  的映射方式中建立，如下面的 ` `Parent.children`` 集合中将会使用 ``list``：


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

        # use a list
        children: Mapped[List["Child"]] = relationship()


    class Child(Base):
        __tablename__ = "child"

        child_id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))

或者在同一个 ``Parent.children`` 中使用集合：


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

        # use a set
        children: Mapped[Set["Child"]] = relationship()


    class Child(Base):
        __tablename__ = "child"

        child_id: Mapped[int] = mapped_column(primary_key=True)
        parent_id: Mapped[int] = mapped_column(ForeignKey("parent.id"))

.. note::  如果使用Python 3.7或3.8，则集合的注释需要使用“typing.List”或“typing.Set”，例如“Mapped[List["Child"]]”或“Mapped[Set["Child"]]”。这些Python版本的“list”和“set”Python内置程序尚不支持泛型注释。例如：


       from typing import List


       class Parent(Base):
           __tablename__ = "parent"

           parent_id: Mapped[int] = mapped_column(primary_key=True)

           # use a List, Python 3.8 and earlier
           children: Mapped[List["Child"]] = relationship()

在使用不带有   :class:`_orm.Mapped`  的映射方式时，比如在使用   :ref:` imperative mappings <orm_imperative_mapping>`  或未定义类型的 Python 代码中以及某些特殊情况下，可以通过  :paramref:`_orm.relationship.collection_class`  参数来直接指定   :func:` _orm.relationship`  的集合类，如下例所示：


    # non-annotated mapping


    class Parent(Base):
        __tablename__ = "parent"

        parent_id = mapped_column(Integer, primary_key=True)

        children = relationship("Child", collection_class=set)


    class Child(Base):
        __tablename__ = "child"

        child_id = mapped_column(Integer, primary_key=True)
        parent_id = mapped_column(ForeignKey("parent.id"))


在缺少  :paramref:`_orm.relationship.collection_class`  或   :class:` _orm.Mapped`  的情况下，缺省集合类型为 ``list``。除了内建的 ``list`` 和 ``set`` 之外，还支持两个字典的变化，在   :ref:`orm_dictionary_collection`  以下章节中有详细描述。任何任意可变序列类型都可以设置为目标集合，需要进行一些额外的配置操作，这将在   :ref:` orm_custom_collection`  一节中进行描述。


.. _orm_dictionary_collection:

字典集合
~~~~~~~~~~~~~~~~~~~~~~

使用字典作为集合需要一些额外的细节处理。这是因为对象总是作为列表从数据库中被加载，并且必须提供一种键生成策略，以便正确地填充字典。   :func:`.attribute_keyed_dict`  函数是实现简单字典集合的最常见方法。它会生成一个字典类，该类将某个映射类的特定属性作为键。下面我们映射了包含“Note”条目字典的“Item”类，这个字典键控制在“Note.keyword”属性中。当使用   :func:` .attribute_keyed_dict`  时，   :class:`_orm.Mapped`  可以使用   :class:` _orm.KeyFuncDict`  或普通的“dict”，如下面的示例所示。不过，在这种情况下，需要在配置时传入  :paramref:`_orm.relationship.collection_class`  参数，以便正确地指定   :func:` .attribute_keyed_dict` ：


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

``Item.notes`` 现在是一个字典：


    >>> item = Item()
    >>> item.notes["a"] = Note("a", "atext")
    >>> item.notes.items()
    {'a': <__main__.Note object at 0x2eaaf0>}

  :func:`.attribute_keyed_dict` ` Note`` 的“keyword”属性都符合字典中的键。例如，当分配给 ``Item.notes`` 时，我们提供的字典键必须与实际``Note``对象的键匹配：


    item = Item()
    item.notes = {
        "a": Note("a", "atext"),
        "b": Note("b", "btext"),
    }

*attribute_keyed_dict* 使用的属性作为键不需要精确映射。使用一个常规的 Python ``@property`` 就可以返回一个键，而不需要关系映射变量身负什么。例如下文，我们将属性映射为 tuple ``Note.keyword`` 和 ``Note.text`` 字段的前十个字母，就是通过这个 tuple 值作为键：


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

以上添加了一个“Note.item”关系，具有双向性的  :paramref:`_orm.relationship.back_populates`  配置。在反向关系上进行分配，就将“Note”添加到“Item.notes”字典中，键将会自动生成：


    >>> item = Item()
    >>> n1 = Note("a", "atext")
    >>> n1.item = item
    >>> item.notes
    {('a', 'atext'): <__main__.Note object at 0x2eaaf0>}

其它内建的字典类型包括   :func:`.column_keyed_dict`  和   :func:` .mapped_collection` ，这是一种任意序列类型的扩展，在一些额外的配置步骤中可以设置为目标集合。这个在   :ref:`orm_custom_collection`  中进行了说明。


.. _key_collections_mutations:

处理键值变化和记录反向关系字典
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

在使用   :func:`.attribute_keyed_dict`  时，字典的 "key" 是从目标对象的某个属性上提取的。**不会跟踪更改此键的情况**。这意味着，在第一次使用时必须分配键值，如果键值改变，集合将不会发生变化。一个常见的例子是在依赖于反向引用将属性映射集合中的值填充时。给定以下内容：

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

以上，如果我们创建了一个“B()”对象，它引用特定的“a()”，则反向引用将会将“B()”添加到“a.bs”集合中，但是如果“B.data”的值尚未设置，则键会为“None”：

    >>> a1 = A()
    >>> b1 = B(a=a1)
    >>> a1.bs
    {None: <test3.B object at 0x7f7b1023ef70>}

在事后设置“b1.data”不会更新集合：

    >>> b1.data = "the key"
    >>> a1.bs
    {None: <test3.B object at 0x7f7b1023ef70>}

如果在构造函数中也尝试设置“B()”，则会看到这一点。参数顺序的不同会导致不同的结果：

    >>> B(a=a1, data="the key")
    <test3.B object at 0x7f7b10114280>
    >>> a1.bs
    {None: <test3.B object at 0x7f7b10114280>}

与

    >>> B(data="the key", a=a1)
    <test3.B object at 0x7f7b10114340>
    >>> a1.bs
    {'the key': <test3.B object at 0x7f7b10114340>}

如果反向引用被用来这样填充属性, 确保使用带有“__init__”方法来正确地按照顺序填充属性。

例如下面的事件处理程序也可以用于跟踪集合的更改：

    from sqlalchemy import event
    from sqlalchemy.orm import attributes


    @event.listens_for(B.data, "set")
    def set_item(obj, value, previous, initiator):
        if obj.a is not None:
            previous = None if previous == attributes.NO_VALUE else previous
            obj.a.bs[value] = obj
            obj.a.bs.pop(previous)

.. _orm_custom_collection:

自定义集合实现
---------------------------------

您也可以使用自己的类型来进行集合的实现。对于简单情况来说，继承 ``list`` 或 ``set``，添加定制行为即可。在其他情况下，则需要用特殊的装饰器告诉 SQLAlchemy 更多关于集合的操作方式。


.. topic:: 是否需要自定义集合实现？

   在大多数情况下，不需要自定义集合实现。自定义集合的最常见用例是验证或转换传入的值，例如将字符串转换为类实例，或者表现出在某些方面内部的数据的不同形式，这些形式对外面呈现出了一个“视图”。

   对于第一个用例，最简单的拦截所有情况的方法是使用   :func:`_orm.validates`  装饰器进行验证和简单转换。有关此内容的示例，请参阅   :ref:` simple_validators` 。


   对于第二个用例，   :ref:`associationproxy_toplevel`  扩展是一种经过测试、广泛使用的系统，它提供了一个读/写的对一个集合的“视图”，包含一个目标对象的属性。因为目标对象属性可以是一个返回任何东西的 ` `@property``，所以使用一些函数就可以构建集合的一系列“替代”视图。

   这种方法使底层映射的集合不受影响，并避免了在需要逐个修改方法时仔细选择集合行为的需要。这些方法当然可以与以上两种方法相结合使用。

集合在 SQLAlchemy 中是透明地进行了 *instrumentation*（工具化）。工具化意味着将集合的常规操作进行跟踪，操作正常情况下会在执行刷新时写入到数据库。此外，集合操作可能会触发 *事件*，这表明需要进行某些次要操作。这样的次要操作包括将子项保存在父   :class:`~sqlalchemy.orm.session.Session`  中（例如，“save-update”级联），以及同步双向关系的状态（如   :func:` .backref` ）。

集合包可以了解列表、集合和字典的基本接口，并自动应用到这些内置类型及其子类中。实现一个基本集合接口的对象派生类型通过 duck-typing 得到检测并进行工具化：

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

``append``、``remove`` 和 ``extend`` 是 ``list`` 的已知成员，并将被自动工具化。
``__iter__`` 不是一个修改器方法，并且不会工具化。 ``foo`` 也不会被工具化。

自然推断（即猜测）并不是牢靠的，当您提供一个 ``__emulates__`` 类属性时，就可以为您实现的接口提供更明细的信息：

.. sourcecode:: python

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

此类与 Python 的 ``list`` 类似（即 “list-like”），因为它有一个 ``append`` 方法，但是 ``__emulates__`` 属性强制将其视为 ``set``。因为 ``remove`` 已知是 Set 接口的一部分，因此将自动工具化。 ``__iter__`` 是默认方法，将用于移除和迭代。默认方法也可以更改：

.. sourcecode:: python+sql

    from sqlalchemy.orm import collection


    class MyList(list):
        @collection.remover
        def zark(self, item):
            # do something special...
            ...

        @collection.iterator
        def hey_use_this_instead_for_iteration(self):
            ...

并不需要完全类似于“列表”或“集合”，只要具有附加器、移除器和迭代器接口标记为 SQLAlchemy 使用即可。追加和移除方法将以映射实体作为唯一参数进行调用，而迭代方法将不带参数调用，并且必须返回一个迭代器。

.. _dictionary_collections:

自定义基于字典的集合
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

  :class:`.KeyFuncDict`  类可以用作自定义类型的基类，也可以作为 mix-in，以便将 ` `dict`` 集合支持快速添加到其他类中。它使用键函数来委托到 ``__setitem__`` 和 ``__delitem__``：


    from sqlalchemy.orm.collections import KeyFuncDict


    class MyNodeMap(KeyFuncDict):
        """Holds 'Node' objects, keyed by the 'name' attribute."""

        def __init__(self, *args, **kw):
            super().__init__(keyfunc=lambda node: node.name)
            dict.__init__(self, *args, **kw)

当继承   :class:`.KeyFuncDict`  时，需要使用装饰器  :meth:` .collection.internally_instrumented`  标记用户定义的版本 of ``__setitem__()`` 或 ``__delitem__()``，**如果** 它们调用了这些已经在   :class:`.KeyFuncDict`  上的方法。这是因为   :class:` .KeyFuncDict`  上的方法已经被工具化了，如果在已经工具化的调用中再次调用它们可能会导致事件被重复触发或不适当地触发，导致在罕见情况下可能会发生内部状态损坏：


    from sqlalchemy.orm.collections import KeyFuncDict, collection


    class MyKeyFuncDict(KeyFuncDict):
        """当你的方法从已经工具化的方法再次调用时，使用 @internally_instrumented"""

        @collection.internally_instrumented
        def __setitem__(self, key, value, _sa_initiator=None):
            # do something with key, value
            super(MyKeyFuncDict, self).__setitem__(key, value, _sa_initiator)

        @collection.internally_instrumented
        def __delitem__(self, key, _sa_initiator=None):
            # do something with key
            super(MyKeyFuncDict, self).__delitem__(key, _sa_initiator)

ORM 理解了 ``dict`` 接口时就像列表和集合一样，并且如果您选择继承 ``dict`` 或通过 duck-typed 类提供类似字典的集合行为，则会自动工具化所有"dict-like" 方法。但是必须装饰加入方法和删除方法。基本字典接口中没有兼容的方法供 SQLAlchemy 去使用。迭代将通过 ``values()`` 进行，除非另有装饰。

工具化和自定义类型
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

许多自定义类型和现有库类可以用作实体集合类型，因为它们已经有了基本集合接口。但是，重要的是要注意，工具化过程将修改类型，自动添加装饰器。这些装饰器是轻量级的，除了关系之外没有作用，但它们会在其他地方触发时增加不必要的开销。当使用库类作为集合时，使用“微不足道的子类”技巧是一个好习惯，将装饰器限制在关系的使用中。例如：


    class MyAwesomeList(some.great.library.AwesomeList):
        pass


    # ... relationship(..., collection_class=MyAwesomeList)

ORM 使用这种方法来处理内置的集合，当直接使用 ``list``、 ``set`` 或 ``dict`` 时，会悄替换为一个微不足道的子类。


集合 API
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